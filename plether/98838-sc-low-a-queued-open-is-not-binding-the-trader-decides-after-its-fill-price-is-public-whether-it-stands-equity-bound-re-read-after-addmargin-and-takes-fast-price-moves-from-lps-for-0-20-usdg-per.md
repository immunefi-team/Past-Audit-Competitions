# \#98838 [SC-Low] A queued open is not binding: the trader decides after its fill price is public whether it stands (equity bound re-read after addMargin) and takes fast price moves from LPs for 0.20 USDG per attempt

**Submitted on Nov 10th 2026 at 14:04:10 UTC by @Tekmer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98838
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Plether's delayed-order model rests on queued orders being binding. Among its protections, packages/perps/SECURITY.md (Delayed-order model) lists: "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options". A live order fills at the first unique Pyth tick after its commit second, so its fill price is fixed, and public, before any keeper executes it.

The order's own execution bounds are not fixed in the same way. `minPostPositionEquityAssets` and `maxPostLeverageBps` are evaluated at execution against the position's live margin, and `CfdEngine.addMargin` lets the owner raise that margin at any time, including while an increase is queued. A trader can therefore queue an increase that fails its own bound by default. If nothing else happens, execution ends the order as a terminal `ConstraintViolation` and the trader pays only the open bounty, at most 0.20 USDG. If the trader calls `addMargin` before the execution transaction, the same order fills at the already-fixed price.

So the trader decides whether a queued order stands, and decides after its fill price is known. Holding one such increase on each side from two accounts, the trader lets the side that is in profit at the fixed fill price stand, lets the other fail, and closes the filled leg at the next tick. LP principal pays the move.

This is the "risk free option" of the pre-audit finding SC-M4 ("Risk Free Option in 60Sec Oracle Window", fixed) through a lever that fix did not cover. The fix pinned the fill tick and made failed opens pay the bounty; the trader can still choose the outcome after the price is known. The settlement-balance bound already has the needed protection (ACCOUNTING_SPEC.md, "Funded settlement minimum for a pending flat open": funded at commit and re-checked on every withdrawal). The equity and leverage bounds do not.

This does not rely on the documented Arbitrum/Pyth clock limitation: the commit is earlier than the fill tick in both clocks, and a fixed commit delay would not stop it.

## How long the choice stays open (measured on the Arbitrum Sepolia v2.0.0 deployment)

The choice lasts from the moment the fill tick is public until a keeper executes the order. The deployed system records both times in every `OrderFinalized` receipt of the `OrderLifecycleBook` (0x327d5fd04e17d96dd5c4f845a0b4256a9e19645c). From deployment to 10 November 2026 13:47 UTC (blocks 325,136,139 to 327,888,696) there are 276 terminal orders, all executed by one address (0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B, the deployer), 268 of them in Live mode. For those 268:

- `receipt.oraclePublishTime - receipt.timing.commitTimestamp` is 1 s in every case: the fill tick is the first second after the commit.
- `terminalTime - receipt.oraclePublishTime` (execution block time minus the fill tick's publish time) is 0 s in 4 orders, 1 s in 32, 2 s in 38, 3 s in 38, 4 s in 29, 5 s in 47, 6 s in 40, 7 s in 31 and 8 s in 9. The median is 4 s, and it is at least 1 s in 264 of 268 (98.5%).
- The keeper became faster on 9 November. For the 66 orders of 9 and 10 November the median is 1 s, and 62 of 66 are at least 1 s.

Timestamps are whole seconds. Pyth publishes several updates per second and Arbitrum produces a block about every 0.25 s, so in almost every recorded execution a later price was public, with time to act on it, before the order was executed.

The PoC includes the opposite case as a control: when a keeper executes in the fill tick's own second, no order stands and the trader loses only the bounties. Keepers are permissionless, so a faster keeper narrows the window; nothing in the contracts closes it.

## Root cause (a82d4fef, packages/perps/src)

- `CfdOrderPolicyEvaluatorBase.sol` L367 and L530-539: post-position margin, equity and leverage are taken from the live position at execution and compared with the order's bounds. Nothing is recorded at commit.
- `CfdEngine.sol` L501-529: `addMargin` checks the caller, a nonzero amount and an open position. It has no restriction while the account has a queued open or increase.
- `OrderRouterExecutionSidecar.sol` L879-887 and `router/OrderRouterBase.sol` L72-74: a bound failure is terminal for the order at the cost of the open bounty (1 bp, capped at 0.20 USDG), whatever the notional.
- `PletherOracle.sol` L610-613: the fill tick is the first unique update published after the commit second, so the fill price is fixed before execution.

## Numbers (PoC, Arbitrum Sepolia v2.0.0 release parameters, 10M pool: 2M Junior, 8M Senior)

Each window: two increases of 3.9M USDG at 500x, one per side, from two accounts. "Trader net" is after every fee, bounty, VPI and carry, with everything withdrawn.

| Case | Orders that stood | Trader net (USDG) | LP principal (USDG) |
|---|---|---|---|
| Keeper executes 4 s after the fill tick (the measured median); basket moves +15 bps | 1 of 2 | +2,721.85 | -4,286.85 |
| Same, basket moves +40 bps | 1 of 2 | +12,467.94 | -14,032.95 |
| Keeper executes 1 s after the fill tick; +15 bps | 1 of 2 | +2,721.85 | -4,286.85 |
| Control: another keeper executes in the fill tick's own second; +15 bps | 0 of 2 | -4.01 | +0.00 |
| Seven windows of ordinary moves (all under 2 bps), keeper 4 s after the fill tick | 0 of 14 | -6.40 | +0.00 |
| Seven windows with moves of +35, -60 and +120 bps, keeper 1 s after the fill tick | 3 of 14 | +74,362.65 | -79,053.79 |
| Same seven windows with 2 bps Pyth confidence on every tick | 3 of 14 | +73,193.18 | -77,884.40 |
| Control for the seven windows: keeper executes in the fill tick's own second | 0 of 14 | -6.42 | +0.00 |

In every test the trader's gain, the LP change, the treasury fees and the keeper bounties sum to zero within 5 USDG (asserted).

## Attack steps

1. Two accounts each deposit collateral and open a small base position, so that `addMargin` is available.
2. Before a moment when the basket may jump, each account commits an increase at 500x whose `minPostPositionEquityAssets` is above what the order itself brings.
3. The first Pyth tick after the commit second fixes the fill price of both orders.
4. Before the keeper's execution transaction, the trader compares the newest price with that fill price. If the move exceeds costs (about 8 bps: two 4 bps execution fees), it calls `addMargin` on the account whose side is in profit.
5. A keeper executes both orders with the fill tick. The validated one fills at the old price. The other ends as `ConstraintViolation` for 0.20 USDG.
6. The trader closes the filled leg at the next tick and repeats. It carries price risk only between seeing the price and that close.

## Impact

The trader's gain is paid from LP principal (Junior first). It is not ordinary trading profit: the trader chose whether to hold the position after seeing prices later than its fill price. Per event the pool loses about the move minus the fees, times the order notional, and one order can be as large as the skew limit allows (39% of pool depth in the PoC). Waiting costs 0.40 USDG per pair of orders. Ordinary one-second moves are below the fees, so the trader uses it when the basket can move fast, for example at scheduled data releases.

## Recommendation

Evaluate the equity and leverage bounds against the margin recorded at commit plus the order's own margin; or reject `addMargin` while the account has a queued open or increase (as the funded settlement minimum already constrains withdrawals); or charge a failed bound on an open a fee proportional to its notional.

## Proof of Concept

Three files in `packages/perps/test/lp/`: `ReleaseMarketHarness.sol` (shared harness), `SelectiveFillAtKnownPrice.t.sol` (the attack over seven windows, and its control) and `SelectiveFillKeeperDelay.t.sol` (keeper delay and a competing keeper). They build on the repository's own `test/perps/BasePerpTest.sol`, install the Arbitrum Sepolia v2.0.0 release parameters through the documented 72 h owner proposals, and use only public entry points as ordinary EOAs (two trader accounts, LPs, keepers): `OrderRouter.commitOrder(OrderRequest)`, `executeOrder`, `CfdEngine.addMargin`, `OrderRouter.settleLpEpoch`. No in-scope contract is modified or impersonated. Pyth and the settlement token are the repository's mocks.

Run from `packages/perps`:

```
FOUNDRY_TEST=test/lp forge test --offline --skip script --match-path 'test/lp/SelectiveFill*.t.sol' -vv
```

Expected: 11 passed (3 in the first suite; the second suite runs its 5 tests and the 3 it inherits).

- `test_keeperFourSecondsAfterFillTick_15bpsMove`: trader +2,721.85 USDG; LP principal -4,286.85.
- `test_keeperFourSecondsAfterFillTick_40bpsMove`: trader +12,467.94; LP principal -14,032.95.
- `test_keeperOneSecondAfterFillTick_15bpsMove`: trader +2,721.85; LP principal -4,286.85.
- `test_control_anotherKeeperExecutesInFillSecond`: no order stands; trader -4.01; LP principal unchanged.
- `test_ordinaryMoves_noOrderStands`: no order stands; trader -6.40; LP principal unchanged.
- `test_traderValidatesOnlyWinningOrders`: trader +74,362.65 after all fees, bounties, VPI and carry; Junior principal 1,999,915.76 -> 1,920,861.22.
- `test_traderValidatesOnlyWinningOrders_withConfidence`: the same with 2 bps Pyth confidence on every tick; trader +73,193.18.
- `test_control_noLaterPriceKnown`: the keeper executes in the fill tick's own second; nothing is validated; trader -6.42; LP principal unchanged.

The keeper delay quoted in the description can be reproduced from the `OrderFinalized` events of the deployed `OrderLifecycleBook` (0x327d5fd04e17d96dd5c4f845a0b4256a9e19645c on Arbitrum Sepolia, from block 323658447): for each receipt with `executionMode == Live`, compare `terminalTime` with `receipt.oraclePublishTime` and `receipt.timing.commitTimestamp`.

### File 1: `packages/perps/test/lp/ReleaseMarketHarness.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Shared scenario harness: Arbitrum Sepolia v2.0.0 release economics, LP funding through the asynchronous vaults,
// trading through the production order entry point, and Pyth ticks supplied as explicit signed-update stand-ins
// (price, confidence, publish time, previous publish time) through the repo's Pyth mock.
//
// Every action is an ordinary EOA calling a public entry point. The only owner actions are the documented 72 h
// configuration proposals that install the release values. No in-scope contract address is impersonated.

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

abstract contract ReleaseMarketHarness is BasePerpTest {

    address internal juniorLp = address(0x1A11);
    address internal seniorLp = address(0xA11CE);
    address internal keeper = address(0x4EE9);

    // Monday 2024-03-11 00:30 UTC: FX market open, outside the Friday risk window.
    uint256 internal constant START = 1_710_117_000;
    uint256 internal constant P0 = 1e8;

    /// Aggregate Pyth confidence attached to every tick, in units of the 8-decimal price.
    uint64 internal tickConf;
    uint256 internal orderNonce;

    // --- Arbitrum Sepolia v2.0.0 release economics (manifest "economics") ---

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

    function _juniorDeposit() internal pure virtual returns (uint256) {
        return 2_000_000e6;
    }

    function _seniorDeposit() internal pure virtual returns (uint256) {
        return 8_000_000e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public virtual override {
        super.setUp();
        // Owner's documented 72 h configuration path: release senior cap/share, opening minimum, adverse multiplier.
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1000e6;
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(routerConfig);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        if (vm.getBlockTimestamp() < START) {
            vm.warp(START);
        }

        // LP deposits through the asynchronous vault flow; a keeper activates them at the epoch boundary.
        // Junior first: Senior admission is capped at 80% of the activated pool.
        uint256 r1 = _requestDeposit(juniorVault, juniorLp, _juniorDeposit());
        vm.warp(juniorVault.depositEpochStart(r1));
        _keeperSettle(P0);
        _claimDeposit(juniorVault, juniorLp, r1);
        uint256 r2 = _requestDeposit(seniorVault, seniorLp, _seniorDeposit());
        vm.warp(seniorVault.depositEpochStart(r2));
        _keeperSettle(P0);
        _claimDeposit(seniorVault, seniorLp, r2);

        // Ten minutes into an epoch, past the post-activation cooldown.
        vm.warp((vm.getBlockTimestamp() / 3600 + 1) * 3600 + 600);
        _keeperRefresh(P0);
    }

    // ------------------------------------------------------------------
    // Pyth ticks: one per second, each with its own publish time and previous publish time
    // ------------------------------------------------------------------

    /// Advances one block and one second and returns the Pyth update published in that second.
    function _nextTick(
        uint256 price
    ) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        return _tickAt(price, vm.getBlockTimestamp());
    }

    /// The Pyth update with publish time `publishTime` (previous update one second earlier).
    function _tickAt(
        uint256 price,
        uint256 publishTime
    ) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: SafeCast.toInt64(SafeCast.toInt256(price)),
                    conf: tickConf,
                    expo: int32(-8),
                    publishTime: publishTime,
                    prevPublishTime: publishTime - 1
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _keeperRefresh(
        uint256 price
    ) internal {
        bytes[] memory data = _nextTick(price);
        vm.prank(keeper);
        router.updateMarkPrice(data);
    }

    function _keeperSettle(
        uint256 price
    ) internal {
        bytes[] memory data = _nextTick(price);
        vm.prank(keeper);
        router.settleLpEpoch(data);
    }

    // ------------------------------------------------------------------
    // LPs and traders: ordinary EOAs through public entry points
    // ------------------------------------------------------------------

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        address lp,
        uint256 requestId
    ) internal {
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _depositCollateral(
        address account,
        uint256 amount
    ) internal {
        settlementToken.mint(account, amount);
        vm.startPrank(account);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(account, amount);
        vm.stopPrank();
    }

    function _withdrawAll(
        address account
    ) internal {
        uint256 balance = clearinghouse.balanceAssets(account);
        if (balance > 0) {
            vm.prank(account);
            clearinghouse.withdraw(account, balance);
        }
    }

    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 id) {
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, ++orderNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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
        id = OrderRouter(address(router)).commitOrder(request);
    }

    /// A keeper executes order `id` with the first tick published after its commit second.
    function _execute(
        uint64 id,
        uint256 price,
        address executor
    ) internal {
        bytes[] memory data = _nextTick(price);
        vm.prank(executor);
        router.executeOrder(id, data);
    }

    /// Commit at the current second, executed one second later at `price` by the ordinary keeper.
    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        uint64 id = _commit(account, side, size, margin, isClose);
        _execute(id, price, keeper);
        assertEq(router.nextExecuteId(), 0, "order queue drained");
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _pledge(
        address account
    ) internal view returns (uint256 margin) {
        (, margin,,,,,) = engine.positions(account);
    }

    /// Position size for `notional` USDG at price `price` (one lot is 100 tokens).
    function _sizeFor(
        uint256 notional,
        uint256 price
    ) internal pure returns (uint256) {
        return (notional / price) * CfdTypes.SIZE_QUANTUM;
    }

    function _trancheNav() internal view returns (uint256 senior, uint256 junior) {
        (senior, junior) = pool.getPendingDepositTrancheState();
    }

}
```

### File 2: `packages/perps/test/lp/SelectiveFillAtKnownPrice.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// A queued open is supposed to be a binding intent: it fills at the first Pyth tick after its commit second and the
// trader cannot cancel it. But the trader's own `minPostPositionEquityAssets` bound is evaluated at execution against
// the position's live pledge, and `CfdEngine.addMargin` lets the trader raise that pledge at any time. So the trader
// commits an increase that fails its own bound by default (cost: the 0.20 USDG execution bounty), waits until the
// fill tick is public, and only then decides whether the order should stand, by adding margin or not.
//
// The trader keeps one such order pending on each side. After a move, the side that is already in profit at the
// fixed fill price is validated; the other side fails. The filled leg is closed at the next tick.
//
// Every action goes through public entry points as an ordinary EOA (two trader accounts, a keeper, LPs). The only
// owner actions are the documented 72 h configuration proposals that install the Arbitrum Sepolia v2.0.0 release
// values. No in-scope contract address is impersonated. Pyth and USDG are the repo's mocks.

import {ReleaseMarketHarness} from "./ReleaseMarketHarness.sol";
import {CfdMath} from "@plether/perps/CfdMath.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

contract SelectiveFillAtKnownPriceTest is ReleaseMarketHarness {

    address internal longAccount = address(0xA77A);
    address internal shortAccount = address(0xA77B);

    uint256 internal constant BASE_NOTIONAL = 2000e6; // small standing position, so that addMargin is available
    uint256 internal constant BASE_MARGIN = 200e6;
    uint256 internal constant SIZE_NOTIONAL = 3_900_000e6; // 39% of pool depth (skew cap 40%)
    uint256 internal constant OPT_IN_MARGIN = 13_000e6;
    uint256 internal constant THRESHOLD_BPS = 10; // round-trip execution fees are 8 bps

    uint256 internal price = P0;
    uint256 internal attempts;
    uint256 internal fills;

    struct Totals {
        int256 traderNet;
        int256 seniorChange;
        int256 juniorChange;
        uint256 treasuryFees;
        uint256 keeperPay;
    }

    // ------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------

    /// The keeper's execution lands one second after the fill tick's publish second. The trader has seen one more
    /// tick by then and validates only the order that is already in profit at the fixed fill price.
    function test_traderValidatesOnlyWinningOrders() public {
        Totals memory t = _runPath(true);
        assertEq(attempts, 7, "seven order pairs committed");
        assertEq(fills, 3, "three orders validated, eleven left to fail");
        assertGt(t.traderNet, 70_000e6, "trader nets > 70,000 USDG after every fee, bounty, VPI and carry");
        assertLt(t.seniorChange + t.juniorChange, -75_000e6, "LP principal pays > 75,000 USDG");
    }

    /// Same attack with a realistic Pyth confidence (2 bps aggregate) on every tick: fills are shifted 0.25 x
    /// confidence against the trader on the way in and on the way out.
    function test_traderValidatesOnlyWinningOrders_withConfidence() public {
        tickConf = 20_000;
        Totals memory t = _runPath(true);
        assertEq(fills, 3, "three orders validated");
        assertGt(t.traderNet, 65_000e6, "trader nets > 65,000 USDG");
        assertLt(t.seniorChange + t.juniorChange, -70_000e6, "LP principal pays > 70,000 USDG");
    }

    /// Control: identical commits and price path, but the keeper executes in the same second the fill tick is
    /// published, so the trader has no later price when it must decide. Nothing is validated; LPs lose nothing.
    function test_control_noLaterPriceKnown() public {
        Totals memory t = _runPath(false);
        assertEq(attempts, 7, "seven order pairs committed");
        assertEq(fills, 0, "no order validated");
        assertLt(t.traderNet, 0, "trader pays bounties and base-position costs");
        assertGt(t.traderNet, -20e6, "and nothing else (under 20 USDG)");
        assertGe(t.seniorChange + t.juniorChange, 0, "LPs do not lose");
    }

    // ------------------------------------------------------------------
    // Scenario: seven one-second windows on one price path (moves in bps of the basket)
    // ------------------------------------------------------------------

    function _runPath(
        bool traderSeesNextTick
    ) internal returns (Totals memory t) {
        _depositCollateral(longAccount, 60_000e6);
        _depositCollateral(shortAccount, 60_000e6);
        uint256 traderStart = _traderValue();
        uint256 treasuryStart = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        uint256 keeperStart = clearinghouse.balanceAssets(keeper);
        (uint256 senior0, uint256 junior0) = _trancheNav();
        console2.log("senior / junior principal before", senior0, junior0);

        _openBase(longAccount, CfdTypes.Side.LONG);
        _openBase(shortAccount, CfdTypes.Side.SHORT);

        int256[7] memory nextTickMoveBps = [int256(0), 35, 2, -60, -3, 120, 6];
        for (uint256 i; i < nextTickMoveBps.length; ++i) {
            _window(nextTickMoveBps[i], traderSeesNextTick);
        }

        // Unwind the two small base positions and withdraw everything.
        _trade(longAccount, CfdTypes.Side.LONG, _size(longAccount), 0, true, price);
        _trade(shortAccount, CfdTypes.Side.SHORT, _size(shortAccount), 0, true, price);
        _keeperRefresh(price);
        _withdrawAll(longAccount);
        _withdrawAll(shortAccount);
        assertFalse(engine.degradedMode(), "pool remains solvent: the loss is borne by LP principal");

        (uint256 senior1, uint256 junior1) = _trancheNav();
        t.traderNet = int256(_traderValue()) - int256(traderStart);
        t.seniorChange = int256(senior1) - int256(senior0);
        t.juniorChange = int256(junior1) - int256(junior0);
        t.treasuryFees = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) - treasuryStart;
        t.keeperPay = clearinghouse.balanceAssets(keeper) - keeperStart;
        console2.log("senior / junior principal after ", senior1, junior1);
        console2.log("order pairs committed / orders validated", attempts, fills);
        console2.log("trader net (USDG atoms, both accounts, after all fees, bounties, VPI, carry)");
        console2.logInt(t.traderNet);
        console2.log("LP principal change: senior, junior");
        console2.logInt(t.seniorChange);
        console2.logInt(t.juniorChange);
        console2.log("protocol treasury fees / keeper bounties", t.treasuryFees, t.keeperPay);
        int256 sum = t.traderNet + t.seniorChange + t.juniorChange + int256(t.treasuryFees + t.keeperPay);
        assertLt(sum > 0 ? sum : -sum, 5e6, "trader gain + LP change + fees nets to zero (within 5 USDG)");
    }

    /// One window. Second 0: both orders are committed. Second 1: the fill tick is published at the unchanged price.
    /// Second 2: the next tick prints `moveBps`. The keeper executes both orders with the second-1 tick.
    function _window(
        int256 moveBps,
        bool traderSeesNextTick
    ) internal {
        ++attempts;
        _keeperRefresh(price);
        (uint64 longOrder, uint256 longFloor) = _commitFailingByDefault(longAccount, CfdTypes.Side.LONG);
        (uint64 shortOrder, uint256 shortFloor) = _commitFailingByDefault(shortAccount, CfdTypes.Side.SHORT);
        longFloor;
        shortFloor;

        // Second 1: the first tick after the commit second. This is the price both orders must fill at.
        uint256 fillPrice = price;
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory fillTick = _tickAt(fillPrice, vm.getBlockTimestamp());

        uint256 nextPrice = moveBps >= 0 ? price + uint256(moveBps) * 1e4 : price - uint256(-moveBps) * 1e4;
        address filled;
        if (traderSeesNextTick) {
            // Second 2: the next tick is public before the keeper's transaction lands.
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(vm.getBlockTimestamp() + 1);
            uint256 seen = nextPrice;
            // LONG profits when the basket falls, SHORT when it rises.
            if (seen * 10_000 <= fillPrice * (10_000 - THRESHOLD_BPS)) {
                filled = longAccount;
            } else if (seen * 10_000 >= fillPrice * (10_000 + THRESHOLD_BPS)) {
                filled = shortAccount;
            }
            if (filled != address(0)) {
                vm.prank(filled);
                engine.addMargin(filled, OPT_IN_MARGIN);
            }
        }
        // Otherwise the keeper executes in second 1 itself: the latest price the trader can know is the fill tick.

        uint256 longBefore = _size(longAccount);
        uint256 shortBefore = _size(shortAccount);
        vm.startPrank(keeper);
        router.executeOrder(longOrder, fillTick);
        router.executeOrder(shortOrder, fillTick);
        vm.stopPrank();
        assertEq(router.nextExecuteId(), 0, "both orders are terminal");
        assertEq(_size(longAccount) > longBefore, filled == longAccount, "LONG order stood only if validated");
        assertEq(_size(shortAccount) > shortBefore, filled == shortAccount, "SHORT order stood only if validated");

        price = nextPrice;
        if (filled != address(0)) {
            ++fills;
            CfdTypes.Side side = filled == longAccount ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT;
            // Close everything at the next tick and re-open the small base position for the next window.
            _trade(filled, side, _size(filled), 0, true, price);
            assertEq(_size(filled), 0, "filled leg closed");
            _openBase(filled, side);
        }
    }

    // ------------------------------------------------------------------
    // Helpers
    // ------------------------------------------------------------------

    function _openBase(
        address account,
        CfdTypes.Side side
    ) internal {
        _keeperRefresh(price);
        _trade(account, side, _sizeFor(BASE_NOTIONAL, P0), BASE_MARGIN, false, price);
        assertGt(_size(account), 0, "base position open");
    }

    /// Commits a 500x increase whose `minPostPositionEquityAssets` bound is above what the order itself provides.
    /// Unless the trader adds `OPT_IN_MARGIN` before execution, the order ends as a terminal constraint violation.
    function _commitFailingByDefault(
        address account,
        CfdTypes.Side side
    ) internal returns (uint64 id, uint256 equityFloor) {
        uint256 notional = SIZE_NOTIONAL;
        uint256 size = _sizeFor(notional, P0);
        uint256 pledge = notional * 2001 / 1_000_000; // just above the 20 bps initial margin
        uint256 liquidationReserve = notional * 11 / 10_000; // 10 bps of the fill notional, with room for the move
        uint256 fee = notional * 5 / 10_000; // 4 bps of the fill notional, with room for the move
        uint256 vpi = _skewCost(notional);
        uint256 margin = pledge + liquidationReserve + fee + vpi;
        // Without the opt-in the position's equity is at most the base pledge plus everything this order can turn
        // into pledge: all of `margin` except the 10 bps liquidation reserve (the fee and VPI allowances stay as
        // pledge when the fill happens to reduce skew and is paid a rebate). The floor sits 1,000 USDG above that;
        // the opt-in clears it in every case.
        equityFloor = BASE_MARGIN + margin - notional * 10 / 10_000 + 1_000e6;

        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, ++orderNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: side == CfdTypes.Side.LONG ? 1 : type(uint256).max,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: equityFloor,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(account);
        id = OrderRouter(address(router)).commitOrder(request);
    }

    /// VPI charged for taking the book from flat to `notional` on one side: 0.5 * k * S^2 / depth.
    function _skewCost(
        uint256 notional
    ) internal view returns (uint256) {
        uint256 depth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        (uint256 vpiFactor,,,,,,,,,) = engine.riskParams();
        return CfdMath.getSkewCost(notional + BASE_NOTIONAL, depth, vpiFactor) + 1e6;
    }

    function _traderValue() internal view returns (uint256) {
        return _accountValue(longAccount) + _accountValue(shortAccount);
    }

    function _accountValue(
        address account
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(account) + clearinghouse.balanceAssets(account)
            + engine.traderClaimBalanceAssets(account);
    }

}
```

### File 3: `packages/perps/test/lp/SelectiveFillKeeperDelay.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// How long the trader's choice stays open. A queued increase that fails its own equity bound by default can be made
// to stand, by `CfdEngine.addMargin`, for as long as no keeper has executed it. These tests vary the two things that
// decide the outcome: how many seconds after the fill tick the keeper's execution lands, and whether another keeper
// executes first, in the fill tick's own second.
//
// Same accounts, pool, release parameters and order shape as SelectiveFillAtKnownPrice.t.sol. Moves are given in
// hundredths of a basis point of the basket price.

import {SelectiveFillAtKnownPriceTest} from "./SelectiveFillAtKnownPrice.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

contract SelectiveFillKeeperDelayTest is SelectiveFillAtKnownPriceTest {

    address internal secondKeeper = address(0x5EC0);

    /// Execution lands four seconds after the fill tick. One second after the fill tick the basket moves +15 bps.
    function test_keeperFourSecondsAfterFillTick_15bpsMove() public {
        Totals memory t = _run(_one(1500), 4, false);
        assertEq(fills, 1, "the winning order stood");
        assertGt(t.traderNet, 2500e6, "trader nets > 2,500 USDG");
        assertLt(t.seniorChange + t.juniorChange, -4000e6, "LP principal pays > 4,000 USDG");
    }

    /// Same with a +40 bps move.
    function test_keeperFourSecondsAfterFillTick_40bpsMove() public {
        Totals memory t = _run(_one(4000), 4, false);
        assertEq(fills, 1, "the winning order stood");
        assertGt(t.traderNet, 10_000e6, "trader nets > 10,000 USDG");
        assertLt(t.seniorChange + t.juniorChange, -12_000e6, "LP principal pays > 12,000 USDG");
    }

    /// Execution lands one second after the fill tick; +15 bps move.
    function test_keeperOneSecondAfterFillTick_15bpsMove() public {
        Totals memory t = _run(_one(1500), 1, false);
        assertEq(fills, 1, "the winning order stood");
        assertGt(t.traderNet, 2500e6, "trader nets > 2,500 USDG");
    }

    /// Control: another keeper executes both orders in the fill tick's own second. Both fail their bound before the
    /// trader has seen a later price; the trader pays the two bounties and nothing is taken from LPs.
    function test_control_anotherKeeperExecutesInFillSecond() public {
        Totals memory t = _run(_one(1500), 4, true);
        assertEq(fills, 0, "no order stood");
        assertLt(t.traderNet, 0, "trader pays bounties and base-position costs");
        assertGt(t.traderNet, -20e6, "and nothing else (under 20 USDG)");
        assertGe(t.seniorChange + t.juniorChange, 0, "LPs do not lose");
    }

    /// Seven windows of ordinary one-second moves (all under 2 bps), keeper four seconds after the fill tick: the
    /// trader never opts in and loses only bounties. The choice costs 0.40 USDG per pair while it waits for a move.
    function test_ordinaryMoves_noOrderStands() public {
        int256[] memory moves = new int256[](7);
        moves[0] = 20;
        moves[1] = -30;
        moves[2] = 50;
        moves[3] = -100;
        moves[4] = 10;
        moves[5] = -40;
        moves[6] = 200;
        Totals memory t = _run(moves, 4, false);
        assertEq(fills, 0, "no order stood");
        assertLt(t.traderNet, 0, "trader pays bounties and base-position costs");
        assertGt(t.traderNet, -20e6, "and nothing else (under 20 USDG)");
        assertGe(t.seniorChange + t.juniorChange, 0, "LPs do not lose");
    }

    // ------------------------------------------------------------------

    function _one(
        int256 moveCbps
    ) internal pure returns (int256[] memory moves) {
        moves = new int256[](1);
        moves[0] = moveCbps;
    }

    function _run(
        int256[] memory movesCbps,
        uint256 keeperDelaySeconds,
        bool anotherKeeperFirst
    ) internal returns (Totals memory t) {
        _depositCollateral(longAccount, 60_000e6);
        _depositCollateral(shortAccount, 60_000e6);
        uint256 traderStart = _traderValue();
        uint256 treasuryStart = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        uint256 keeperStart = clearinghouse.balanceAssets(keeper) + clearinghouse.balanceAssets(secondKeeper);
        (uint256 senior0, uint256 junior0) = _trancheNav();

        _openBase(longAccount, CfdTypes.Side.LONG);
        _openBase(shortAccount, CfdTypes.Side.SHORT);
        for (uint256 i; i < movesCbps.length; ++i) {
            _windowWithDelay(movesCbps[i], keeperDelaySeconds, anotherKeeperFirst);
        }
        _trade(longAccount, CfdTypes.Side.LONG, _size(longAccount), 0, true, price);
        _trade(shortAccount, CfdTypes.Side.SHORT, _size(shortAccount), 0, true, price);
        _keeperRefresh(price);
        _withdrawAll(longAccount);
        _withdrawAll(shortAccount);
        assertFalse(engine.degradedMode(), "pool remains solvent: the loss is borne by LP principal");

        (uint256 senior1, uint256 junior1) = _trancheNav();
        t.traderNet = int256(_traderValue()) - int256(traderStart);
        t.seniorChange = int256(senior1) - int256(senior0);
        t.juniorChange = int256(junior1) - int256(junior0);
        t.treasuryFees = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) - treasuryStart;
        t.keeperPay = clearinghouse.balanceAssets(keeper) + clearinghouse.balanceAssets(secondKeeper) - keeperStart;
        console2.log("keeper delay after the fill tick (seconds)", keeperDelaySeconds);
        console2.log("order pairs committed / orders that stood", attempts, fills);
        console2.log("trader net (USDG atoms, both accounts, after all fees, bounties, VPI, carry)");
        console2.logInt(t.traderNet);
        console2.log("LP principal change (senior + junior)");
        console2.logInt(t.seniorChange + t.juniorChange);
        console2.log("protocol treasury fees / keeper bounties", t.treasuryFees, t.keeperPay);
        int256 sum = t.traderNet + t.seniorChange + t.juniorChange + int256(t.treasuryFees + t.keeperPay);
        assertLt(sum > 0 ? sum : -sum, 5e6, "trader gain + LP change + fees nets to zero (within 5 USDG)");
    }

    /// Second 0: both failing-by-default increases are committed. Second 1: the fill tick is published at the
    /// unchanged price. Second 2: the basket moves by `moveCbps` and stays there. Second 1 + `keeperDelaySeconds`:
    /// the keeper's execution lands, carrying the second-1 tick.
    function _windowWithDelay(
        int256 moveCbps,
        uint256 keeperDelaySeconds,
        bool anotherKeeperFirst
    ) internal {
        ++attempts;
        _keeperRefresh(price);
        (uint64 longOrder,) = _commitFailingByDefault(longAccount, CfdTypes.Side.LONG);
        (uint64 shortOrder,) = _commitFailingByDefault(shortAccount, CfdTypes.Side.SHORT);

        uint256 fillPrice = price;
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory fillTick = _tickAt(fillPrice, vm.getBlockTimestamp());
        if (anotherKeeperFirst) {
            vm.startPrank(secondKeeper);
            router.executeOrder(longOrder, fillTick);
            router.executeOrder(shortOrder, fillTick);
            vm.stopPrank();
            assertEq(router.nextExecuteId(), 0, "both orders are terminal before any later price exists");
        }

        uint256 nextPrice = moveCbps >= 0 ? price + uint256(moveCbps) * 1e2 : price - uint256(-moveCbps) * 1e2;
        vm.roll(vm.getBlockNumber() + keeperDelaySeconds);
        vm.warp(vm.getBlockTimestamp() + keeperDelaySeconds);

        // The trader has seen the later price. LONG profits when the basket falls, SHORT when it rises.
        address chosen;
        if (nextPrice * 10_000 <= fillPrice * (10_000 - THRESHOLD_BPS)) {
            chosen = longAccount;
        } else if (nextPrice * 10_000 >= fillPrice * (10_000 + THRESHOLD_BPS)) {
            chosen = shortAccount;
        }
        if (chosen != address(0) && router.nextExecuteId() != 0) {
            vm.prank(chosen);
            engine.addMargin(chosen, OPT_IN_MARGIN);
        }

        uint256 longBefore = _size(longAccount);
        uint256 shortBefore = _size(shortAccount);
        if (!anotherKeeperFirst) {
            vm.startPrank(keeper);
            router.executeOrder(longOrder, fillTick);
            router.executeOrder(shortOrder, fillTick);
            vm.stopPrank();
        }
        assertEq(router.nextExecuteId(), 0, "both orders are terminal");
        bool longStood = _size(longAccount) > longBefore;
        bool shortStood = _size(shortAccount) > shortBefore;
        assertEq(longStood, chosen == longAccount && !anotherKeeperFirst, "LONG order stood only if validated");
        assertEq(shortStood, chosen == shortAccount && !anotherKeeperFirst, "SHORT order stood only if validated");

        price = nextPrice;
        if (longStood || shortStood) {
            ++fills;
            address account = longStood ? longAccount : shortAccount;
            CfdTypes.Side side = longStood ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT;
            _trade(account, side, _size(account), 0, true, price);
            assertEq(_size(account), 0, "filled leg closed");
            _openBase(account, side);
        }
    }

}
```
