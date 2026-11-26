# \#97620 [SC-Critical] Booked VPI charge is not reserved against its later rebate: a junior LP can exit at a higher share price and re-enter

**Submitted on Nov 2nd 2026 at 14:41:17 UTC by @drMurlly for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97620
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

In plain terms (code read, with the test result below): the pool counts a fee as LP profit the moment it is paid, although it can hand much of it back when the trade closes; an LP who leaves in between is paid for profit that the LPs who stay would later give back.

Terms. VPI is virtual price impact, the skew-based charge or rebate on a trade (`packages/perps/ACCOUNTING_SPEC.md:52`). NAV is the pool net asset value that prices LP shares. Skew is long open interest minus short open interest. Junior is the first-loss LP tranche. By code read, F is the full VPI charge booked at the open (F = 0.005 * r^2 * D) and the part that is refunded when the position closes is at most F; inferred from the logged figures, it is about 73 percent of F in the linked test. USDG is the 6-decimal settlement token. Carry is the carry fee the pool collects from open positions. A keeper is any account that executes a queued order. An epoch is the hourly LP settlement round.

Plether Perps books the VPI charge that a trader pays when opening a skew-increasing position as pool cash and LP revenue at once (`packages/perps/src/CfdEngineSettlementSidecar.sol:454-464`). Part of that charge is refundable: when the same position later reduces the skew, the pool pays a rebate out of free pool cash (`packages/perps/src/libraries/CfdEnginePlanLib.sol:973-978`), bounded by the VPI the position actually paid (`packages/perps/src/libraries/CloseAccountingLib.sol:112-118`). The refundable part of booked VPI is therefore cash in the pool and in the junior share price, but the NAV used to price LP redemptions has no term for the possible future rebate. By code read and the test result below, while the position is open the junior share price is too high, relative to what the pool pays out if the position closes by reducing skew, by up to F, the full booked charge (F = 0.005 * r^2 * D), where r is the net skew as a fraction of pool depth D (so F is at most 8 bps of D). In a copy of the linked test with log lines added (supplementary measurements, Part A), the cash actually paid out at the close is 3,279.450138 USDG, against 4,499.9991 USDG booked. By code read, the execution fee and carry due on the close are netted against the rebate, so the pool keeps the netted fee (Attack steps, fourth item); the 73 percent figure is inferred from the logged figures, and F is an upper bound.

By code read, a junior LP who calls `requestRedeem` while an unrelated trader's skew is open would be paid at the higher price, and would re-enter after the trader closes. The junior LPs who stayed in the pool pay the rebate. This is a value transfer between LPs: in the test (inferred from the Delta table) about 99 percent of the exiting LP's gain is the incumbent junior LP's loss and about 1 percent comes from the trader. The trader receives less cash at the close in the attack world than in the passive world (13th item of Conditions and limits). By code read, it needs no oracle manipulation, no flash capital, no privileged role and no collusion with the trader; every call it makes is permissionless. By code read of the test and the vault, the attacker must be a large existing junior LP (6,000,000 USDG in the test), must wait a one-hour cooldown between entering and exiting, and cannot choose when the trader opens or closes (5th item of Conditions and limits).

In the linked test the same market is run twice from one EVM snapshot: in the PASSIVE world the large junior LP (the attacker) does nothing, in the ATTACK world it exits and re-enters as described. Compared with the passive world, the attacker ends +1,967.669907 USDG, the other junior LP (the incumbent) ends -1,948.864416 USDG and the trader ends -18.805002 USDG. By code read of the test file: the incumbent deposited 4,000,000 USDG, the risk parameters come from the Arbitrum Sepolia v2.0.0 release manifest, the EVM is in-process and the synthetic mark is 1.00. Inferred from the separate runs, the effect is bounded and conditional: profitable only above about 8 percent net skew, at most 8 bps of pool depth per cycle, a possible loss with several traders, and, in one separate run with the contract-default frozen fee, a loss while the oracle is frozen (11th item of Conditions and limits). The test uses a junior-only pool, a mock oracle with a constant mark, a test-only router adapter for the trader's commit and testnet manifest parameters; the test was not run on a fork or a deployment. Details: "Conditions and limits" (under Vulnerability Details). The full Delta table is in the Proof of Concept field.

## Vulnerability Details

### Root cause

By inspection, all code below is in `packages/perps/src` at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. Line numbers were checked against that commit.

#### Site A: the VPI charge is booked as LP revenue at open

`CfdEngineSettlementSidecar.executeOpen` charges the trade cost, which includes the VPI charge, and records the cash that reaches the pool (after `clearinghouse.applyOpenCost`) as claimant `Revenue` with `CashArrived`. `packages/perps/src/CfdEngineSettlementSidecar.sol:454-464`:

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

`HousePool.recordClaimantInflow` (`packages/perps/src/HousePool.sol:624-651`) adds the amount to `accountedAssets`. At the next reconcile the increase is allocated by the waterfall to junior principal after any senior impairment is restored (`packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:84-98`, `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:147-158`). By code read, from that moment the junior share price includes the whole VPI charge.

#### Site B: part of the charge is paid back at close, from pool cash

When the position later reduces the skew, `packages/perps/src/libraries/CfdEnginePlanLib.sol:973-978` plans an action rebate and pays it from free pool cash. It is waived only when that cash is not there:

```solidity
        if (delta.actionRebateAssets > 0) {
            delta.actionRebatePaidAssets =
                delta.actionRebateAssets < freePoolCashAssets ? delta.actionRebateAssets : freePoolCashAssets;
            delta.actionRebateWaivedAssets = delta.actionRebateAssets - delta.actionRebatePaidAssets;
            poolCashAfterCollectionsAssets -= delta.actionRebatePaidAssets;
        }
```

`packages/perps/src/CfdEngineSettlementSidecar.sol:621-624` pays it with `IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets)`.

`packages/perps/src/libraries/CloseAccountingLib.sol:112-118` bounds the rebate by the VPI the position paid over its lifetime:

```solidity
        // Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
        // where an attacker opens at high depth, reduces pool depth, then closes to extract a
        // net-negative lifetime VPI rebate. ...
        if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
            state.vpiDeltaAssets = -state.proportionalAccrualAssets;
        }
```

The VPI charge is refundable on the paths where the position later reduces skew at the same or lower depth, the pool has free cash, and the position is closed rather than liquidated: then the lifetime clamp returns up to what the position paid (`packages/perps/src/libraries/CloseAccountingLib.sol:103-118`, `packages/perps/src/libraries/CfdEnginePlanLib.sol:973-978`). On those paths the pool pays out cash that the share price already counted as revenue (`packages/perps/src/libraries/CfdEnginePlanLib.sol:973-978`, `packages/perps/src/CfdEngineSettlementSidecar.sol:454-464`). On other paths (liquidation, which does not compute a fresh VPI delta, `packages/perps/SECURITY.md:1436`; depth growth, where the rebate shrinks; an unfunded rebate, which is waived) the pool keeps it. By code read, the NAV used for exits has no term for the refund on the paths where it is paid, so the exit price is too high exactly there; on the paths where the pool keeps the charge, the NAV is already right (an LP who exits before a liquidation gives up the pool's gain, 8th item of Conditions and limits).

#### Where the NAV omits it

LP entry and exit are priced from `HousePoolAccountingLib.buildReconcileSnapshot` (`packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78`). It starts from the pool's physical assets (`terminalAssets = engineSnapshot.physicalAssets`), which already contain the cash from Site A, adds the marked price delta `terminalLpPriceDeltaAssets` and subtracts trader claims. The inputs come from `CfdEngineProtocolLens.getHousePoolInputSnapshot` (`packages/perps/src/CfdEngineProtocolLens.sol:62-70`): `snapshot.physicalAssets = poolAssets`, where `poolAssets = engineContract.pool().totalAssets()`, and `snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets`. `terminalLpPriceDeltaAssets` is the marked price PnL of open positions. `TerminalNavBookV2` deliberately excludes VPI from it, and the documentation states this as the intended design (`packages/perps/SECURITY.md:863`, `packages/perps/SECURITY.md:871-872`, `packages/perps/ACCOUNTING_SPEC.md:1064`). The documents say that "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV" (`packages/perps/SECURITY.md:872`). This report accepts that exclusion: it does not ask for anticipated VPI to be added to the price book. It concerns a different quantity, the VPI already collected, which the documents do not address: the booked positive VPI is already cash in `physicalAssets` (`packages/perps/WHITEPAPER.md:411-412`, in the paragraph on the open-time admission test) while the rebate that returns part of it is paid from the same pool. The suggested fix therefore changes the documented NAV treatment, a design decision for the maintainers; the evidence that it is a defect rather than a trade-off is the LP-to-LP transfer in the test below.

### Size of the unreserved amount

The VPI curve is `CfdMath.tryGetSkewCost` (`packages/perps/src/CfdMath.sol:118-140`): cost(S, D) = vpiFactor * S^2 / (2 * D). By code read, with vpiFactor = 0.01 (Arbitrum Sepolia v2.0.0 release manifest, `deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json:166`, also `script/DeployPerpsArbitrumSepolia.s.sol:434-445`; the test sets it in `_riskParams()`) and S = r * D:

    F = cost(S, D) = 0.005 * r^2 * D   (inferred from 0.01 / 2 = 0.005)

`D` is the pool's economic depth, canonical assets minus outstanding trader claims (`packages/perps/SECURITY.md:391`). `r` is the net skew divided by that total depth (junior plus senior). By code read of the manifest, `maxSkewRatio` is 0.4e18, so r is inferred to be at most 0.4 and F to be at most 0.0008 * D, 8 bps of depth. Mainnet launch parameters were not available to us; F scales linearly with vpiFactor and quadratically with skew, so the figures here would change with them.

By the formula above (inferred), at the fixture (r = 0.30, D = 10,000,000 USDG) F is 4,500 USDG. By code read of the test file, the attacker holds 60 percent of the junior tranche, so its share of F is inferred to be about 2,700 USDG; it deposited 6,000,000 USDG. Its wallet ends at 6,002,699.999140 USDG (Delta table in the Proof of Concept field). The passive LP is inferred to earn 732.329233 USDG of pool revenue over the window the attacker is out of the pool; by the same inference, 2,699.999140 - 732.329233 = 1,967.669907, the gain in the Delta table. That 732.329233 USDG is inferred to be 60 percent of the part of the booked charge that the pool keeps in the passive world: by the same inference, 4,499.9991 booked less 3,279.450138 paid at the close is 1,220.548962, and 0.6 times that is 732.329377, within 0.0002 USDG of the figure in the Delta table (the booked and paid amounts come from a copy of the test with log lines added; see Attack steps, fourth item).

Equivalently, the attacker's gain is inferred to be its 60 percent share of the rebate the pool pays: by the same inference, 0.6 * 3,279.450138 = 1,967.670083 against 1,967.669907 in the Delta table. In the control with the VPI charge switched off the attacker loses 11.080668 USDG over the same window, which is the carry yield it forgoes (`test_control_noVpi`). Inferred from the wallet balances in the Delta table, with the VPI charge it forgoes 732.329233 USDG of revenue and its wallet gains 2,699.999140 USDG of unreserved charge. By code read, with no VPI there is no rebate to net the close execution fee against, so the fee goes to the treasury and the pool keeps only carry; with the VPI charge, inferred from the logged figures, about 1,200 USDG of the 1,220.55 USDG the pool keeps is that fee (Attack steps, fourth item).

### Attack steps

By code read of the test file, all steps are calls by accounts with no role. The trader is an unrelated user the attacker neither controls nor funds. Times are seconds after the trader's open in the linked test.

1. By code read of the test file: the trader commits an order (in the test through `LegacyOrderRouterHarness`, a test-only commit adapter, see the 9th item of Conditions and limits) and a keeper (anyone) executes it, a LONG of 3,000,000 tokens at mark 1.00 (30 percent of depth). The VPI charge is booked as LP revenue (Site A). By code read of the test file, it is executed at +1 s.
2. By code read of the test file, the attacker, a junior LP holding 60 percent of the junior shares, calls `requestRedeem` on the junior vault for all its shares at +600 s. A request must be placed at least 5 minutes before the epoch boundary (`packages/perps/src/TrancheVault.sol:39`); epochs are one hour (`packages/perps/src/HousePool.sol:178`).
3. By code read, anyone calls `OrderRouter.settleLpEpoch` (`packages/perps/src/OrderRouter.sol:479-484`, permissionless) after the boundary; in the test, by code read of the test file, at +3,599 s. By code read, the attacker is paid at the NAV that includes F while the position is still open. Inferred from the logged pool depths, the attacker's exit cuts pool depth from about 10M to about 4M. Inferred from those depths, after the exit the 3,000,000 token position is about 75 percent of the remaining depth, above the 40 percent maxSkewRatio; the redemption was nonetheless paid in full in the test (code read of the test file). `maxSkewRatio` is checked when a position is opened or increased (`packages/perps/src/libraries/CfdEnginePlanLib.sol:473-477`; `packages/perps/src/CfdTypes.sol:88-94`); we did not find an equivalent check on LP exits, but we did not audit that. We report this as a fact of the test and make no separate claim about it.
4. By code read of the test file, the trader closes (requested at +14,400 s, executed at +14,401 s). The close plan computes the action rebate as the VPI rebate (clamped to the lifetime VPI accrued) less the execution fee and carry due on the close (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1044` and `packages/perps/src/libraries/CfdEnginePlanLib.sol:1059-1061`), and the pool pays it from its cash (Site B). Measured in a copy of the test with log lines added (output in the linked supplementary measurements file), the booked charge is 4,499.9991 USDG and the pool pays 3,279.450138 USDG at the close in the passive world and 3,260.645136 USDG in the attack world. The difference to the booked charge (1,220.55 and 1,239.35 USDG, inferred from the logged figures) stays in the pool. In this path the execution fee is not collected from the trader and credited to the treasury; it is netted against the rebate, so the pool pays less and keeps it (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1044`, `packages/perps/src/libraries/CfdEnginePlanLib.sol:1059-1061`). The treasury balance did not change at the close in either world (see the supplementary measurements). The WHITEPAPER passage cited below states that the protocol execution fee belongs to treasury rather than LP assets for the open; we did not check the netted close path against that intent. About 1,200 USDG of each difference is, by inference, consistent with the 4 bps execution fee (1,200 USDG on 3,000,000 notional, manifest executionFeeBps 4) and the remainder with carry netted in the same plan; we did not log those components separately. Inferred from the logged depths, not from a clamp flag: by the VPI formula the cost of closing is about 4,497.98 USDG at the passive world's depth (10,004,501.999 USDG), below the 4,499.9991 accrued, so the lifetime clamp does not bind there; at the attack world's depth (4,001,801.999 USDG) the cost is about 11,244.9 USDG and the clamp binds. The gross VPI rebate is therefore inferred to be about 2.02 USDG larger in the attack world (0.1 percent of the attacker's gain). The cash paid is nevertheless 18.805002 USDG lower. The amount netted against the rebate is inferred to be about 1,239.35 USDG in the attack world against about 1,218.52 in the passive world (these two figures and the clamp reading are computed from the logged figures, not logged themselves). We did not determine why the netted amount differs. The remaining junior LPs bear the rebate.
5. The attacker calls `requestDeposit` with the cash it received and claims the shares after the next epoch (+17,999 s, by code read of the test file). By code read, because the refundable amount is no longer in the share price, it would receive more shares for the same cash than it gave up. A one-hour holder cooldown (`packages/perps/src/TrancheVault.sol:38`) applies before its next exit.

In the output the attacker holds 6,004,946.797644148 shares at the end instead of 6,000,054.000432003, for the same cash.

### Conditions and limits

Numbers marked "separate runs" come from a parametrised copy of the test (net skew, attacker share, senior deposit and position life are its knobs) or from the same test with the fix sketch applied. That copy is not among the linked files; its raw output lines are copied into the linked file `supplementary-measurements.txt`. These numbers give context and are not needed for the finding.

1. Collusion-free LP cycle. The report covers this case and nothing else. The trader is unrelated; the attacker is a plain junior LP. A variant in which the LP also operates the trader was not found to be reliably profitable once a senior tranche is present and is not part of this report.
2. Skew threshold. Separate runs (not run again for this report): the cycle loses 0.30 USDG at 5 percent net skew and 0.77 USDG at 8 percent, and gains 58.77 USDG at 10 percent; the break-even lies between 8 and 10 percent (a linear read of the 8 and 10 percent rows gives about 8 percent). Inferred from those runs, above 20 percent the gain is close to s * D * r * (0.005 * r - 4.05e-4), with s the attacker's share of the junior tranche (within 0.3 percent at 20, 30 and 39 percent). By the same inference, that fit does not hold at 5 to 10 percent (it gives -46.5 at 5 percent, -2.4 at 8 percent and +57.0 at 10 percent), and the 4.05e-4 term is of the size of the 4 bps execution fee (manifest executionFeeBps 4) plus carry, which the close plan nets against the rebate; if that is its source, the break-even is where 0.005 * r = 4.05e-4, near 8 percent net skew. We did not test that attribution separately.
3. Bounded. Inferred from the manifest's maxSkewRatio, F is at most 8 bps of pool depth. Inferred from the Delta table and the separate runs, per cycle the gain is about 3.3 bps of the attacker's capital at 30 percent skew and about 6.0 bps at 39 percent (3,604.70 USDG in a separate run).
4. Netting traders. The gain follows the net skew, and with several traders the sign depends on the closing order. Separate runs (not run again for this report): with a long and a short trader, net skew 20 percent with the smaller leg closing first gave the attacker +466.75 USDG; closing the dominant leg first, or a net skew of zero, gave a loss of between 1,331.83 and 5,420.96 USDG; the reason, by inference, is that the attacker's exit cuts depth and every later skew-increasing leg then pays VPI at the smaller depth.
5. Timing. The position must be open at an hourly settlement. Separate run (not run again for this report): with the trader closing 30 minutes after the open, before the first epoch boundary, the attacker gained 0.000006 USDG.
6. By code read of the test file, the headline is the 4 hour case. Separate runs (not run again for this report): the attacker still gained at longer position lives (24 h +1,912.10, 72 h +1,778.72, 168 h +1,511.96 USDG), but part of that gain is carry-collection timing, a documented collection-time ownership policy (`packages/perps/ACCOUNTING_SPEC.md:731-740`). This report does not rely on them.
7. Fixture and senior tranche. By code read of the test file, the test has a junior-only pool: the incumbent holds 4,000,000 USDG, the attacker 6,000,000 USDG, and the senior tranche is only its 1 USDG seed. By code read of the test file, there is one trader with a 3,000,000 token LONG (r = 30 percent) and no offsetting flow. A pool with a real senior tranche is not modelled. By code read, two code paths may move the result either way: F accrues to the junior tranche only while it is sized on total depth, and junior redemption funding is limited by the senior covenant (`packages/perps/SECURITY.md:396`; `packages/perps/src/HousePool.sol:1205-1216`), which can reduce or block the exit step. Separate runs (not run again for this report): with a senior tranche added (the same 3,000,000 token position), the gain was +1,181.79 USDG with 5,000,000 senior and +843.41 with 10,000,000 senior, against +1,967.67 with none. Separate run (not run again for this report): with 30,000,000 senior the exit seems to have been capped by the covenant: the attacker's first exit paid 2,500,098 USDG of its 6,000,000 USDG, and the cycle lost 2,496,678 USDG against the passive LP. Earlier sweep rows (not run again for this report): the senior deposit is set to 0.25 to 3 times the junior tranche, and the rows showed gains between +1,730.50 and +4,244.13 USDG without the sketch (supplementary file, B2b); we did not reconcile those rows with the 5, 10 and 30 million rows above (not established), so the senior effect is not shown to be monotone. The launch senior ratio is unknown to us and we make no claim about either direction for a real pool.
8. Oracle. By code read of the test file, the test uses `MockPyth` with confidence 0 and a constant basket mark of 1.00; price PnL between the exit and the close is not modelled. In two separate runs (not run again for this report) with the mark moved by +1 percent and -1 percent between the open and the close the gain stayed positive (+1,961.69 and +1,921.16 USDG). In one separate run (not run again for this report) in which the trader was liquidated after the exit (mark +6 percent, margin 1 percent) the cycle lost 13,679.999150 USDG against the passive LP; the reason, by inference, is that the exiting LP forgoes the pool's gain on the liquidation, so the cycle carries price risk. We did not model a price distribution, real oracle updates with signatures or confidence.
9. Router. The trader's commit uses `LegacyOrderRouterHarness`, a test-only adapter that exposes a `commitOrder` entry with wide execution bounds. The harness is the repository's own (`test/utils/LegacyOrderRouterHarness.sol`, deployed by `BasePerpTest`): it turns the scalar commit into a V3 order request whose execution bounds are all set to the maximum and enters the production V3 commit by self-delegatecall (`packages/perps/test/utils/LegacyOrderRouterHarness.sol:63-81`). Execution (`executeOrder`), VPI computation and LP settlement (`settleLpEpoch`) are the production code paths. The production commit is the financially bounded V3 `OrderRouter.commitOrder` (`packages/perps/src/OrderRouter.sol:77-90`); we did not run it with realistic bounds, and the repository's production-router fixture (`test/perps/gas/DirectCloseGas.t.sol`) was not used.
10. Set-up. `setUp` warps time backward once, after the timelock warps (as the repository's own test base does); harmless here. By code read of the test file, the deployer acts only in `setUp`; the exploit leg uses no privileged call.
11. The report concerns a live oracle. When the oracle is frozen, exits pay a frozen-window LP fee (`packages/perps/src/HousePool.sol:1147-1155`). Separate run (not run again for this report): with the oracle frozen and a frozen LP fee that is not a release-manifest value (it is of the order of the contract's own default of 75 bps for junior), the attacker was paid 5,959,529 USDG for shares worth 6,000,000 USDG, a haircut of about 67 bps, far above the 8 bps maximum. Mainnet values of that fee are not known to us.
12. Fees. The manifest sets the junior maintenance fee to 0 bps (`script/DeployPerpsArbitrumSepolia.s.sol:138`, `deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json:192`). Other values were not re-run.
13. The trader ends 18.805002 USDG lower in the attack world. This equals the difference between the cash the pool pays the trader at the close in the passive and attack worlds (3,279.450138 against 3,260.645136 USDG; Attack steps, fourth item). The gross VPI rebate is inferred to be about 2.02 USDG larger in the attack world through the clamp (Attack steps, fourth item), so the cash difference is the amount netted against the rebate (about 20.83 USDG more in the attack world) less that 2.02 USDG; we did not determine why the netted amount differs. Inferred from the Delta table, about 1 percent of the attacker's gain comes from the trader and 99 percent from the incumbent junior LP.
14. Not run on a fork or deployment. The ticks are synthetic. The launch TVL, the senior share, the open interest and the mainnet risk parameters are unknown to us.

### Suggested fix

Treat the refundable part of booked VPI as a liability until the position closes. The sketch in the linked `suggested-fix.diff` adds `totalVpiAccrued` to `CfdEngine`, the sum of every open position's `vpiAccrued`, updated in `settlementWritePosition` (`packages/perps/src/CfdEngine.sol:1269`) and `settlementDeletePosition` (`packages/perps/src/CfdEngine.sol:1283`). `CfdEngineProtocolLens` then subtracts it from `terminalLpPriceDeltaAssets` before the snapshot reaches `buildReconcileSnapshot` (`packages/perps/src/CfdEngineProtocolLens.sol:70`).

Tested effect, on a copy of the same commit: `test_exploit_collusionFreeExitTiming` fails (attacker 5,999,999.999950 USDG against 6,000,732.329233 for the passive LP, a delta of -732.329283). The controls still pass (5 passed, 1 failed in the mutant log). Separate runs (not run again for this report): with the sketch applied, the single-trader cycle rows (net skew 5 to 39 percent, longer position lives, and six senior-tranche rows with the senior deposit set to 0.25 to 3 times the junior tranche; these are not the 5, 10 and 30 million rows of limit 7, which were not re-run with the sketch) all turned negative; the headline row loses its forgone revenue (about -732.33 USDG). Same separate runs (not run again for this report): the 30-minute-close control stayed at +0.000006 USDG, rows with several traders or a liquidation also stayed negative (losses of 733.92 to 16,379.998 USDG), and the two rows (12a, 12c) in which LP depth grows while a position is open stayed positive for the entering LP (see the side effect below). The `CfdEngine` runtime size went from 23,137 to 23,278 bytes (limit 24,576). `HousePool` is at 24,571 bytes in the same build, so a fix should avoid changing it.

The maintainers should also consider two points. One point: the sketch sums `vpiAccrued` including negative values; how negative balances should be treated, given the dedicated reserve for them (`packages/perps/SECURITY.md:866`), was not tested. Another point: the rebate is computed at close-time depth and clamped to the accrued amount (`packages/perps/src/libraries/CloseAccountingLib.sol:103-118`). If LP deposits grow depth while a position is open, the rebate shrinks and the difference stays in the pool. Today the staying LPs keep it; with the sketch the entering LP would. One separate row (not run again for this report): depth growing from 5,000,000 to 10,000,000 USDG while a 1,500,000 notional position was open; the trader ended 1,120.79 USDG lower, against +3.44 with vpiFactor 0, so the effect is VPI-related; the rebate itself was not logged. This is a side effect to decide together with the fix, and it is adjacent to the AI-scan items named below.

### Why this is not a known issue

The programme lists the audit reports and `SECURITY.md` Known Limitations as known. We checked the programme-listed items (the answers are in `OUT-OF-SCOPE-ANSWERS.md`, one by one). By inspection, no item lists LP exit pricing against booked VPI as a limitation. We address the nearest documents at the top of this list.

- **Anticipated VPI outside terminal NAV** (`packages/perps/SECURITY.md:863`, `packages/perps/SECURITY.md:871-872`, `packages/perps/ACCOUNTING_SPEC.md:1064`, `packages/perps/WHITEPAPER.md:411-412`). This is the nearest documented item. It says the NAV book does not add anticipated VPI as an asset, and that a new close-time rebate is paid only from free pool cash and is waived if unfunded (`packages/perps/SECURITY.md:871-872`). The finding concerns the other side: the charge is already in `physicalAssets` as cash (`packages/perps/WHITEPAPER.md:411-412` "Positive VPI increases physical pool assets", in the paragraph on the open-time admission test), the rebate is paid in full whenever cash is free, and the waiver applies only when cash is short (`packages/perps/src/libraries/CfdEnginePlanLib.sol:973-978`), so it does not protect the staying LPs. The documentation states a symmetry principle for LP entry and exit ("an entrant receives the same discount for inherited marked liabilities that an exiting LP bears", `packages/perps/ACCOUNTING_SPEC.md:172`; `packages/perps/SECURITY.md:859-860`). By inspection, those sentences are about marked trader price PnL, so we claim no literal contradiction, only that the behaviour reported here runs against their purpose. The documentation reserves the negative-VPI side (`packages/perps/SECURITY.md:866`) and has no counterpart for the positive side.
- **Carry collection timing** (`packages/perps/ACCOUNTING_SPEC.md:731-740`, `packages/perps/WHITEPAPER.md:974-977`). A stated policy about who owns carry revenue that is collected as cash and never returned (`packages/perps/ACCOUNTING_SPEC.md:731-740`). The VPI charge differs: where the refund applies it is paid back out of the pool (`packages/perps/src/libraries/CfdEnginePlanLib.sol:973-978`), so the timing transfer here is of money the pool will later pay out, not of revenue it keeps.
- **AI scan items "Temporary pool liquidity can reduce a skew-increasing close charge" and "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate"** (`audits/scan-plether-fi-plether-core-d4541b49.md:1693` and `audits/scan-plether-fi-plether-core-d4541b49.md:1742`; the second says at `audits/scan-plether-fi-plether-core-d4541b49.md:1796` that "positive accrued VPI can therefore be reclaimed at a rate selected through current depth", and at `audits/scan-plether-fi-plether-core-d4541b49.md:1808` that a manipulated negative VPI "becomes an actual pool-funded action rebate whenever free cash is available"). These items establish that the rebate is pool-funded and can be steered through depth, by a single actor changing depth around its own close. By inspection, none of these concerns what an LP is paid while the charge is open. Here the depth change is an ordinary epoch-queued LP redemption, and the cash rebate is not enlarged: the pool pays the trader 18.805002 USDG less in the attack world (3,260.645136 against 3,279.450138 USDG). Inferred from the logged depths: the gross VPI rebate before fee and carry is about 2.02 USDG larger because the clamp binds at the smaller depth, which is 0.1 percent of the gain, so the attacker's gain does not come from an amplified rebate. By code read, it comes from the junior share price paid to the redeemer while the refundable VPI is unreserved. The clamp comment at `packages/perps/src/libraries/CloseAccountingLib.sol:112-114` describes an "LP sandwich" in which one actor opens at high depth, reduces pool depth, and closes to extract a net-negative lifetime VPI rebate. By code read, that is a single actor enlarging its own rebate; here the cash rebate is not enlarged (the gross VPI rebate is inferred to grow by about 2 USDG), and the transfer comes from the share price paid to the exiting LP.
- **VPI limitations** (`packages/perps/SECURITY.md:1434-1444`). The bullets (`packages/perps/SECURITY.md:1434-1444`) cover zero-depth opens, protective closes, the lifetime clamp on rebates (`packages/perps/SECURITY.md:1443`), the frozen-market spread and the partial-close approximation. By inspection, no bullet concerns LP pricing while a charge is open.
- **Known Limitations, LP accounting** (`packages/perps/SECURITY.md:1380-1399`). The bullet at `packages/perps/SECURITY.md:1384` says a positive marked receivable is not cash, so share NAV can exceed redeemable liquidity. That concerns uncollected trader losses, not VPI cash that is already in the pool.
- **Audit reports.** We found no row in the audit reports in the repository's `audits` folder about the NAV omitting refundable booked VPI or about LP exit timing relative to booked VPI. We also searched the text of the collaborative audit report (2026-05-12) and the Cantina review in that folder for "vpi", "rebate" and "virtual price": no matches. The closest item is SC-H1 in the pre-audit report ("Deposit Before Yield is Accrued Allows to Dilute Yield", impact "loss of funds for previous depositors", listed as High and addressed by a senior rewrite). By inspection, it treats LP entry timing against revenue not yet recognised in the vault as a loss for earlier depositors; this report is a different mechanism (a VPI charge booked as revenue that the pool later pays back), on the same kind of timing theme, and we do not claim the SC-H1 fix covers it.
- **Out-of-scope clause "basic economic and governance attacks".** Read as covering attacks that work through market power or capital, the clause seems to concern 51 percent control, price manipulation, flash loans and vote buying. By code read of the test file, this report uses none of them: no price is moved, no borrowed capital is used, no vote is cast and no privileged role acts. The transfer comes from a deterministic inconsistency between two code paths the protocol runs itself: the VPI charge is booked as pool cash (`packages/perps/src/CfdEngineSettlementSidecar.sol:454-464`) and the rebate is paid from that cash (`packages/perps/src/libraries/CfdEnginePlanLib.sol:973-978`), while the LP exit price counts only the first. It is demonstrated to the micro-USDG by a runnable test (see the Delta table) and, by inference from the manifest, bounded at 8 bps of pool depth per cycle (Brief/Intro).

## Impact Details

### Impact

By code read and the Delta table: junior LPs who stay in the pool lose principal, not only yield: the incumbent ends at 3,998,539.355063 USDG against a 4,000,000 USDG deposit. By code read, the cause is a permissionless sequence of ordinary calls that needs no role, no oracle manipulation and no flash capital; the attacker must be a large existing junior LP (60 percent of the junior tranche, 6,000,000 USDG, in the test). Inferred from the manifest and the separate runs, the loss is bounded (at most 8 bps of pool depth per cycle) and conditional (net skew above about 8 percent), and it is a transfer between LPs (about 99 percent of the gain), not a theft from the protocol. We select Medium, "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)", as the programme row for a bounded, permissionless harm to users. The attacker here does profit, so the row's example wording does not describe the attacker but the damage; the severity is for the judge to set. By the Delta table (inferred), because the incumbent's balance falls below its deposit, value is lost in the test, which is why we do not select the row for returns that are not delivered without a loss of value.

### Impact in the test

Source: `test_exploit_collusionFreeExitTiming`. By code read of the test file, both worlds start from one EVM snapshot, run the same trader schedule and end with both LPs fully redeemed to USDG, so every row is a real ERC20 balance. Attack world minus passive world, in USDG: attacker +1,967.669907; incumbent junior LP -1,948.864416; trader -18.805002; pool raw balance -0.000489; clearinghouse custody 0; token supply 0. The holder deltas sum to 0. The full Delta table is in the Proof of Concept field.

By arithmetic on the Delta table (inferred), the attacker's gain equals the incumbent's loss plus the trader's loss plus 0.000489 USDG of pool dust; nothing is created. By code read and the Delta table (inferred): the incumbent ends below its 4,000,000 USDG deposit in the attack world (3,998,539.355063), a loss of principal and not only of yield; in the passive world it earns 488.219479 USDG over the same window.

## References

- Code under review: `packages/perps/src` at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 (repository `immunefi-team/audit-comp-plether`). Principal files cited (paths under `packages/perps/src`): `CfdEngineSettlementSidecar.sol:454-464`, `CfdEngineSettlementSidecar.sol:621-624`; `libraries/CfdEnginePlanLib.sol:473-477`, `libraries/CfdEnginePlanLib.sol:973-978`, `libraries/CfdEnginePlanLib.sol:1041-1044`, `libraries/CfdEnginePlanLib.sol:1059-1061`; `libraries/CloseAccountingLib.sol:103-118`; `CfdMath.sol:118-140`; `HousePool.sol:624-651`; `libraries/HousePoolAccountingLib.sol:60-78`; `libraries/HousePoolReconcilePlanLib.sol:84-98`; `libraries/HousePoolWaterfallAccountingLib.sol:147-158`; `CfdEngineProtocolLens.sol:62-70`; `OrderRouter.sol:77-90`, `OrderRouter.sol:479-484`; `TrancheVault.sol:38-39`; `CfdEngine.sol:1269`, `CfdEngine.sol:1283`.
- Repository documents cited (paths under `packages/perps`): `SECURITY.md:391`, `SECURITY.md:396`, `SECURITY.md:859-860`, `SECURITY.md:863`, `SECURITY.md:866`, `SECURITY.md:871-872`, `SECURITY.md:1380-1399`, `SECURITY.md:1434-1444`; `ACCOUNTING_SPEC.md:52`, `ACCOUNTING_SPEC.md:172`, `ACCOUNTING_SPEC.md:731-740`, `ACCOUNTING_SPEC.md:1064`; `WHITEPAPER.md:411-412`, `WHITEPAPER.md:974-977`.
- Release manifest for the risk parameters: `deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json:166`, `deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json:192`, `script/DeployPerpsArbitrumSepolia.s.sol:138` and `script/DeployPerpsArbitrumSepolia.s.sol:434-445`.
- Prior art checked (details in "Why this is not a known issue"): AI scan `audits/scan-plether-fi-plether-core-d4541b49.md:1693`, `audits/scan-plether-fi-plether-core-d4541b49.md:1742`, `audits/scan-plether-fi-plether-core-d4541b49.md:1796`, `audits/scan-plether-fi-plether-core-d4541b49.md:1808`; pre-audit report finding SC-H1 and SC-M3; the collaborative audit report (2026-05-12) and the Cantina review (searched for "vpi", "rebate", "virtual price": no matches).
- Proof of concept, run logs, the fix sketch `suggested-fix.diff`, `usdg-readout.txt` (a generated USDG view of the logs) and `supplementary-measurements.txt` (the booked charge, the cash rebate at the close, and the raw lines of the "separate runs" figures in this report): linked in the Proof of Concept field.


## Link to Proof of Concept

https://gist.github.com/e8a8b17170599b8607e4e76fda29e1fa

## Proof of Concept

## Proof of Concept

Proof of concept: a Foundry test on the in-process EVM (no fork, no deployment), at the contest commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. By code read of the test file, the oracle ticks are synthetic (`MockPyth`, constant basket mark 1.00).

### What the test does

1. By code read of the test file, `setUp` builds the full stack from the repository's own test base, with the risk parameters, six price feeds and pool configuration of the Arbitrum Sepolia v2.0.0 release manifest applied by override. By code read of the test file, the deployer or owner acts only in `setUp`.
2. By code read of the test file, two worlds are run from one EVM snapshot: ATTACK (LP `A` cycles) and PASSIVE (`A` does nothing). By code read of the test file, everything else, including the trader's schedule and the final time, is identical.
3. By code read of the test file, in both worlds a passive junior LP (`INC`, 4,000,000 USDG) and the attacker `A` (6,000,000 USDG) deposit through `requestDeposit` and claim, and an unrelated trader `T` deposits margin to the clearinghouse.
4. By code read of the test file, `T` opens a 3,000,000 token LONG (30 percent of depth) through the router and a keeper executes it. By code read of the test file: in the ATTACK world `A` calls `requestRedeem` for all its shares 10 minutes later, anyone calls `settleLpEpoch` at the next hourly boundary, `A` claims its cash, `T` closes 4 hours after the open, and `A` calls `requestDeposit` with the cash and claims the shares one epoch later.
5. By code read of the test file: both worlds idle until 10 hours after the open; then `INC` and `A` redeem fully, so every compared figure is a plain USDG wallet balance (`MockSettlementToken`, 6 decimals).
6. By code read of the test file, the test prints the Delta table and asserts: `A` ends above its passive self; `INC` ends below its passive self; `T` does not gain; at least 95 percent of `A`'s gain is `INC`'s loss; `A`'s gain is within 1 percent of 1,967.7 USDG; holder deltas sum to zero and the token supply does not change.

### Reproduction

Requirements: Foundry (forge 1.7.1 was used) and the repository checked out at the contest commit with its submodules. Make `git status` clean before the test file is copied in; afterwards it lists the untracked `packages/perps/test/perps/poc/` directory.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc
cp /path/to/VpiFloatExitTiming.t.sol packages/perps/test/perps/poc/VpiFloatExitTiming.t.sol
cd packages/perps
forge test --match-contract VpiFloatExitTimingTest -vv
```

The repository's default profile is used (optimizer on with 200 runs, `via_ir`, solc 0.8.35). No other file is changed. The recorded output below is from this exact command on a pristine copy of the commit with one new file added, three consecutive runs with identical figures (the header of `run.log` states this; unverified beyond that header). Not established by the attached logs: for the final file in a clean copy with no `out` or `cache` directory, the plain command compiles 194 files and takes about 47 seconds and about 3.3 GB peak memory on a 32-core machine (on a busy machine the compile alone took over two minutes).

Not established by the attached logs: robustness checks on an earlier revision of the test, before the live-balance assertions and the scenario knobs were added, gave identical figures to the micro-USDG: optimizer on and off with `via_ir` on; optimizer off and `via_ir` off; and every `block.timestamp` in the test replaced by `vm.getBlockTimestamp()` (which rules out the compiler caching the timestamp across `vm.warp`). On the final file the exploit test was repeated with the optimizer disabled (`FOUNDRY_OPTIMIZER=false`, `via_ir` on): the Delta table is byte-identical, attacker delta 1967669907 micro-USDG (this repeat is not recorded in the attached logs and is not established by them). The other checks were not repeated on the final file.

### Expected output

Test names, control logs and the Delta table, copied from the recorded run (run 3 of 3). Gas figures and timings can differ slightly between machines; the USDG figures do not. Omitted from this excerpt: blank lines, the line "No files changed, compilation skipped", the third header line of the printed table (a note that the ticks are synthetic and that the run was local, not run against a fork or any deployment), the `IMPACT_METRIC` marker lines from the test and the final summary line of the suite; the suite timing varies per machine. The full recorded output (`run.log`, linked with the test) contains these lines.

```text
Ran 6 tests for test/perps/poc/VpiFloatExitTiming.t.sol:VpiFloatExitTimingTest
[PASS] test_control_falseClaimsAreRejected() (gas: 29601288)
Logs:
  FALSE claim 1 failed as it must: FALSE CONTROL: no-position exit gains > 1 USDG: 5999999999950 <= 6000000999929
  FALSE claim 2 failed as it must: FALSE CONTROL: passive beats passive: 6000732329233 <= 6000732329233
[PASS] test_control_lateExit() (gas: 16644829)
Logs:
  control lateExit: A gain (micro-USDG), signed:
  5
[PASS] test_control_noOpenPosition() (gas: 8422235)
Logs:
  control noOpenPosition: A gain (micro-USDG), signed:
  21
[PASS] test_control_noVpi() (gas: 16940518)
Logs:
  control noVpi: A gain (micro-USDG), signed (negative = forgone carry yield):
  -11080668
[PASS] test_control_passive() (gas: 15254480)
[PASS] test_exploit_collusionFreeExitTiming() (gas: 16838008)
Logs:
  ==================== DELTA TABLE (USDG, 6 dec; attack world minus passive world) ====================
  Both worlds: same snapshot, same T schedule, same end time, INC and A fully redeemed to USDG.
  ROW attacker A wallet
      attack-world  : 6002699999140
      passive-world : 6000732329233
      delta (micro-USDG, attack - passive):
  1967669907
  ROW incumbent junior LP INC wallet
      attack-world  : 3998539355063
      passive-world : 4000488219479
      delta (micro-USDG, attack - passive):
  -1948864416
  ROW background trader T wallet
      attack-world  : 317560446036
      passive-world : 317579251038
      delta (micro-USDG, attack - passive):
  -18805002
  ROW keeper wallet
      attack-world  : 0
      passive-world : 0
      delta (micro-USDG, attack - passive):
  0
  ROW settler wallet
      attack-world  : 0
      passive-world : 0
      delta (micro-USDG, attack - passive):
  0
  ROW HousePool raw USDG balance (pool raw assets)
      attack-world  : 1999761
      passive-world : 2000250
      delta (micro-USDG, attack - passive):
  -489
  ROW vault contracts: senior
      attack-world  : 0
      passive-world : 0
      delta (micro-USDG, attack - passive):
  0
  ROW vault contracts: junior
      attack-world  : 0
      passive-world : 0
      delta (micro-USDG, attack - passive):
  0
  ROW MarginClearinghouse custody total
      attack-world  : 1200200000
      passive-world : 1200200000
      delta (micro-USDG, attack - passive):
  0
  ROW other holders (engine, router, oracle, deployer)
      attack-world  : 0
      passive-world : 0
      delta (micro-USDG, attack - passive):
  0
  MEMO rows (inside the rows above, not added again):
  MEMO pool accountedAssets (accounted assets)
      attack-world  : 1999761
      passive-world : 2000250
      delta (micro-USDG, attack - passive):
  -489
  MEMO clearinghouse sub-account: T
      attack-world  : 0
      passive-world : 0
      delta (micro-USDG, attack - passive):
  0
  MEMO clearinghouse sub-account: keeper
      attack-world  : 200000
      passive-world : 200000
      delta (micro-USDG, attack - passive):
  0
  MEMO clearinghouse sub-account: protocol treasury
      attack-world  : 1200000000
      passive-world : 1200000000
      delta (micro-USDG, attack - passive):
  0
  MEMO A shares held right before the first exit request (attack world)
      attack-world  : 6000054000432003
      passive-world : 6000054000432003
      delta (micro-USDG, attack - passive):
  0
  MEMO A shares held at the end before the final redeem
      attack-world  : 6004946797644148
      passive-world : 6000054000432003
      delta (micro-USDG, attack - passive):
  4892797212145
  ROW settlement token total supply
      attack-world  : 10320002000000
      passive-world : 10320002000000
      delta (micro-USDG, attack - passive):
  0
  SUM of holder deltas (must be 0; = supply delta):
  0
  supply delta:
  0
  ======================================================================================================
Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 28.22ms (98.96ms CPU time)
```

### How to read the controls and the mutant

- `test_exploit_collusionFreeExitTiming` is the finding. It fails if the attacker does not end above the passive world.
- By code read of the test file, `test_control_passive` runs the passive world twice from one snapshot and requires identical balances, which indicates that the harness has no drift.
- `test_control_noOpenPosition`: the attacker exits when no position is open. The gain is +21 micro-USDG, which is numerical noise. Without an open skew there is no refundable amount in the price.
- `test_control_lateExit`: by code read of the test file, the attacker exits only after the trader closed and the rebate was paid. The gain is +5 micro-USDG. The timing of the exit is what matters.
- `test_control_noVpi`: by code read of the test file, `vpiFactor` is set to 0 at set-up. The cycle then loses 11.080668 USDG (the log labels it forgone carry yield). Inferred from the logged figures: with the VPI charge the same forgone revenue is 732.329233 USDG, which is 60 percent of the 1,220.55 USDG of the booked charge that the pool does not pay back (about 1,200 of it is the netted execution fee; Description, "Size of the unreserved amount"). Without the VPI charge there is no gain; the mutant below, which reserves the accrued VPI, removes the gain.
- `test_control_falseClaimsAreRejected` runs two deliberately false assertions (a no-position exit that gains more than 1 USDG; the passive world beating itself). By code read of the test file, the test passes only if both revert (the two logged lines are the evidence), so the assertions used by the finding would discriminate.
- Mutant. Applying `suggested-fix.diff` (reserve the sum of open positions' accrued VPI in the NAV) to the same commit turns the exploit test red while the controls stay green. The attacker then ends 732.329283 USDG below its passive self, which is its forgone LP revenue. To reproduce: from the repository root run `git apply /path/to/suggested-fix.diff`, then `cd packages/perps && forge test --match-contract VpiFloatExitTimingTest -vv`; undo with `git apply -R /path/to/suggested-fix.diff`. Excerpts of the mutant run (`mutant-run.log`, linked with the test; lines between the excerpts are left out):

```text
[FAIL: EXPLOIT: attacker USDG must exceed passive-control USDG: 5999999999950 <= 6000732329233] test_exploit_collusionFreeExitTiming() (gas: 21156332)
```

The attacker's row of the Delta table in the mutant run:

```text
  ROW attacker A wallet
      attack-world  : 5999999999950
      passive-world : 6000732329233
      delta (micro-USDG, attack - passive):
  -732329283
```

The suite result line:

```text
Suite result: FAILED. 5 passed; 1 failed; 0 skipped; finished in 28.99ms (100.06ms CPU time)
```

### Fixture

| Item | Value |
|---|---|
| Commit | a82d4fefb02768e5b3bddcc97dee371eee24fea8, scope `packages/perps/src` |
| Toolchain | forge 1.7.1, repository default profile (optimizer 200, `via_ir`, solc 0.8.35) |
| Junior tranche (code read of the test file) | `INC` 4,000,000 USDG and `A` 6,000,000 USDG (A holds 60 percent), plus a 1 USDG seed |
| Senior tranche (code read of the test file) | the 1 USDG seed alone (no real senior tranche) |
| Trader `T` (code read of the test file) | one account; LONG of 3,000,000 tokens at mark 1.00 (net skew r = 30 percent of depth); 320,000 USDG deposited, 160,000 USDG posted as margin |
| Oracle (code read of the test file) | `MockPyth`, six feeds, confidence 0, constant basket mark 1.00 (synthetic ticks) |
| Risk parameters (code read of the manifest) | Arbitrum Sepolia v2.0.0 release manifest: vpiFactor 0.01e18, maxSkewRatio 0.4e18, maintMarginBps 10, initMarginBps 20, baseCarryBps 500; junior maintenance fee 0 bps. Mainnet launch parameters were not available to us |
| Router (code read) | `LegacyOrderRouterHarness`, the repository's own test adapter (deployed by its `BasePerpTest`): it builds a V3 order request with all execution bounds at the maximum and enters the production V3 commit (`packages/perps/src/OrderRouter.sol:77-90`) by self-delegatecall; `executeOrder` and `settleLpEpoch` are the production functions. Realistic bounds were not run |
| Timeline (code read of the test file) | seconds after T's open: open executed +1; `A` requestRedeem +600; epoch boundary +3,598; settle executed +3,599; close requested +14,400, executed +14,401; `A` requestDeposit +14,401, claimed +17,999; both worlds end +36,001, final exits at the next epoch boundaries |
| Callers (code read of the test file) | `INC`, `A`, `T`, a keeper and a settler: all accounts with no role |
| Set-up caveat | `setUp` moves time backward once after the timelock warps (as the repository's test base does); harmless here |
| Not modelled | price moves between exit and close, a real senior tranche, several traders, real oracle updates with signatures, launch TVL, the production commit surface |

### Sensitivity to net skew

Code read of the test file: attacker gain against the passive LP, D = 10,000,000 USDG junior, `A` = 60 percent, no real senior tranche, mark 1.00. Source: separate runs (not run again for this report) of a parametrised copy of the test with the net skew (basis points of depth) as the changed knob. They are not part of the recorded output above (not run again for this report), except the 30 percent row, which comes from the recorded run. The raw output lines of these rows are in the linked file `supplementary-measurements.txt`, and the rows also appear in at least one further recorded sweep. To check a row yourself, change the constant `R_BPS` in the test (3000 in the recorded run, by code read of the test file; the trader size is the skew times the depth) and run the same command; the file's sha256 then no longer matches.

| Net skew r | Attacker gain (USDG) | Incumbent junior change (USDG) |
|---:|---:|---:|
| 5 percent | -0.30 | +0.85 |
| 8 percent | -0.77 | +2.22 |
| 10 percent | +58.77 | -56.52 |
| 20 percent | +714.84 | -706.02 |
| 30 percent | +1,967.67 | -1,948.86 |
| 39 percent | +3,604.70 | -3,575.20 |

Inferred from the rows above, the break-even lies between 8 and 10 percent net skew; a linear read of the 8 and 10 percent rows gives about 8 percent. Separate runs (not run again for this report): scale is linear in depth, D = 1,000,000 at 39 percent gave +360.56, and D = 100,000,000 at 10 percent gave +587.47.

### Files

The test file `VpiFloatExitTiming.t.sol` (sha256 9de0f840c161e9aafcf6d7af599344185e7284fa3523701403d51e4f1dbc09dc), `run.log`, `mutant-run.log`, the fix sketch `suggested-fix.diff` (applies to the same commit with `git apply`) and `supplementary-measurements.txt` (the booked charge and the cash rebate at the close, taken from a copy of the test with log lines added, and the raw lines of the separate runs quoted in the report) and `usdg-readout.txt` (a generated view of the logs, with the USDG value added after the micro-USDG integers) are in this secret gist:

https://gist.github.com/e8a8b17170599b8607e4e76fda29e1fa
