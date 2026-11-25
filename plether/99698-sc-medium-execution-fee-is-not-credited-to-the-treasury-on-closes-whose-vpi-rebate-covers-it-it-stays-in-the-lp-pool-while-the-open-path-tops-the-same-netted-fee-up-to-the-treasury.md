# \#99698 [SC-Medium] Execution fee is not credited to the treasury on closes whose VPI rebate covers it (it stays in the LP pool), while the open path tops the same netted fee up to the treasury

**Submitted on Nov 14th 2026 at 20:37:13 UTC by @EntropyResearch for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99698
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

**Category:** Insight — Code Optimizations and Enhancements (fee accounting consistency)
**Target:** `packages/perps/src/libraries/CfdEnginePlanLib.sol` (`_planCloseActionSettlement`, close fee top-up), `packages/perps/src/CfdEngineSettlementSidecar.sol` (open and close settlement)
**Commit:** a82d4fefb02768e5b3bddcc97dee371eee24fea8

## Brief/Intro

When a close nets to a VPI rebate (the rebate is at least the 4 bps execution fee plus carry and any frozen spread), the fee is deducted from the rebate the trader receives but is never credited to the protocol treasury: it stays in the LP pool. The open path treats the identical situation differently and tops the netted fee up to the treasury from free pool cash. The trader pays the same either way; only the recipient of the fee changes, depending on whether the rebate-netted fee belongs to an open or a close. No funds are lost or frozen, so this is reported as an Insight.

## Vulnerability Details

1. Close planning nets every action charge into one amount (`CfdEnginePlanLib.sol:1041-1044`):

```solidity
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(
        cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets
    );
```

   If the result is negative (L1059-1060) it becomes `delta.actionRebateAssets`, and `actionChargeWithheldAssets` / `actionChargeCollectedAssets` stay zero.

2. The execution fee recorded for the close is then capped by those zero amounts (`CfdEnginePlanLib.sol:1066-1068`):

```solidity
uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;
delta.executionFeeAssets =
    cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;
```

3. The close top-up is computed from the capped value (`CfdEnginePlanLib.sol:984-987`), so it is zero as well:

```solidity
uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets;
...
delta.protocolFeeTopUpAssets = feeTopUpAssets < freePoolCashAssets ? feeTopUpAssets : freePoolCashAssets;
```

   and close settlement credits only `protocolFeeCreditedAssets + delta.protocolFeeTopUpAssets` to the treasury (`CfdEngineSettlementSidecar.sol:626-628`), i.e. nothing. The rebate actually paid to the trader is `|actionNetAssets|`, already reduced by the full fee, so the fee amount remains in the pool.

4. The open path calls the same helper with the uncapped fee (`CfdEngineSettlementSidecar.sol:498`):

```solidity
_settleProtocolFeeTopUp(host, delta.executionFeeAssets, protocolFeeCreditedAssets);
```

   `_settleProtocolFeeTopUp` (`CfdEngineSettlementSidecar.sol:905-925`) pays the uncredited part of the fee from free pool cash to the treasury, so a skew-reducing open whose rebate covers the fee still credits the treasury.

**What the documentation says.** `CfdEnginePlanTypes.sol:318` documents the close-side cap ("Assessed execution fee capped by eligible withheld price gain and collected action cash"), and the whitepaper charge table (`WHITEPAPER.md:810`) lists the execution fee as "Protocol service revenue on traded notional | Treasury, only when cash credited". The close behaviour therefore matches that wording and may be intended. The point of this insight is that the open path does not follow the same rule (it credits the treasury from pool cash when no cash is credited), and `WHITEPAPER.md:411-413` states that "the protocol execution fee belongs to treasury rather than LP assets". The two settlement paths disagree on who receives the same netted fee.

## Impact Details

- No user or LP funds are lost or frozen; the trader's total cost is identical in both designs.
- On every close that nets to a rebate, the 4 bps execution fee (`executionFeeBps`, 4 at release) moves from protocol revenue to LP income. In the attached test, a $200k position opened and closed at an unchanged price costs the trader both fees (≈160 USDG); the treasury receives only the open fee (80 USDG) and the LP pool receives the close fee.
- When it applies: VPI uses `cost(skew) = vpiFactor × skew² / (2 × depth)` (`CfdMath.sol:118-139`). A close of size `N` from skew `S` returns `vpiFactor × N × (2S − N) / (2 × depth)`, capped at the VPI the position paid (`CloseAccountingLib.sol:111-118`). With release `vpiFactor = 0.01` and a 4 bps fee, the close nets to a rebate whenever `(2S − N) > 8%` of pool depth (ignoring carry), e.g. any position of at least 8% of depth closed into an otherwise flat book, or a small position closed while skew is above ~4% of depth (provided it paid at least that much VPI when opening). Large skew-reducing closes are therefore the typical case, not an edge case.

## Recommendation

Make the two paths consistent. If the fee is meant to be protocol revenue whenever it is assessed, keep `delta.executionFeeAssets` at the assessed fee for closes that net to a rebate and let the existing free-cash-capped top-up (`protocolFeeTopUpAssets` / `_settleProtocolFeeTopUp`) credit the treasury, as the open path does. If the "only when cash credited" rule is the intended policy, apply the same cap on the open path and align `WHITEPAPER.md:411-413`.

## References

- `packages/perps/src/libraries/CfdEnginePlanLib.sol:984-987` (close fee top-up), `1036-1073` (`_planCloseActionSettlement`)
- `packages/perps/src/CfdEngineSettlementSidecar.sol:498` (open top-up with the uncapped fee), `626-628` (close top-up), `905-925` (`_settleProtocolFeeTopUp`)
- `packages/perps/src/CfdEnginePlanTypes.sol:318` (NatSpec of the close cap)
- `packages/perps/src/CfdMath.sol:118-139` (VPI skew cost), `packages/perps/src/libraries/CloseAccountingLib.sol:105-122` (close VPI, lifetime clamp, execution fee)
- `packages/perps/WHITEPAPER.md:411-413, 810`

## Proof of Concept

## Proof of Concept

Runnable code is not required for Insights; a runnable test is included so that every number below can be reproduced.

### Conditions under which this insight is valuable

- Any close whose VPI rebate is at least the execution fee plus pending carry (and the frozen spread, if frozen). With release `vpiFactor = 0.01` and `executionFeeBps = 4`, a close of size `N` from skew `S` nets to a rebate when `(2S - N) > 8%` of pool depth, subject to the lifetime clamp (the rebate cannot exceed the VPI the position paid). Typical cases: a position of at least 8% of depth closed into an otherwise flat book; any skew-reducing close while skew is above ~4% of depth.
- On each such close the 4 bps fee is retained by the LP pool instead of the treasury, while a skew-reducing open in the same situation still credits the treasury. The value of the insight is consistency of protocol fee revenue between the two settlement paths, not recovery of user funds.

### How to run

No RPC, fork or environment variables are needed; the test runs in the repository's own Foundry suite at the commit in scope.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc
# copy the two files below into packages/perps/test/perps/poc/
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/perps/poc/CloseFeeTreasuryInsight.t.sol' -vv
```

Tested with Foundry `forge 1.9.0` (solc 0.8.35 is selected by `foundry.toml`); the first via-IR compile takes roughly 10 minutes. The test uses the repository's `BasePerpTest` fixture (real `CfdEngine`, `MarginClearinghouse`, `HousePool`, `TrancheVault`s, `PletherOracle`, `OrderRouter`; only Pyth and USDG are mocked), release risk parameters (`vpiFactor = 0.01`), a 1,002,000 USDG pool, and production Router entrypoints (`commitOrder`, `executeOrder`). No in-scope contract is modified.

### What the tests show

- `test_insight_closeRebateKeepsExecutionFeeInPool` — a $200k LONG is opened on an empty book (VPI 199.60 + fee 80.00, fee credited to the treasury) and closed at the same price. The close returns a VPI rebate of about 199.6 USDG, reduced by the 80 USDG fee and 0.04 USDG carry, so the trader nets +119.56 USDG; the treasury receives 0. Over the round trip the trader paid 160.04 USDG of fees and carry (plus a 0.20 USDG keeper bounty); the treasury received 80 and the LP pool 80.04.
- `test_insight_openRebateStillCreditsExecutionFee` — control for the open path: after a $200k LONG, a $200k SHORT open is skew-reducing and nets to a rebate (+119.36 USDG to the trader), yet the treasury still receives the full 80 USDG fee, paid from pool cash (pool outflow 199.56 USDG).

### Test output (`-vv`)

```
[PASS] test_insight_closeRebateKeepsExecutionFeeInPool() (gas: 6262976)
Logs:
  STEP 1: Alice opens a $200k LONG on an empty book (skew-increasing: pays VPI + 4 bps fee)
    trader cost (USDG, negative = net receipt): 279.800798
    LP pool inflow (USDG): 199.600798
    treasury inflow (USDG): 80.000000
    keeper execution bounty (USDG): 0.200000
  STEP 2: Alice closes the same position at the same price (skew-reducing: VPI rebate > fee)
    trader cost (USDG, negative = net receipt): -119.560985
    LP pool inflow (USDG): -119.560985
    treasury inflow (USDG): 0.000000
    keeper execution bounty (USDG): 0.000000
  RESULT: round trip at an unchanged price (VPI nets out)
    trader round-trip cost (USDG): 160.239813
    treasury total (USDG): 80.000000
    LP pool total (USDG): 80.039813
    keeper execution bounties total (USDG): 0.200000
[PASS] test_insight_openRebateStillCreditsExecutionFee() (gas: 7136670)
Logs:
  STEP 1: Alice opens a $200k LONG (creates long skew)
  STEP 2: Bob opens a $200k SHORT (skew-reducing open: VPI rebate > fee)
    trader cost (USDG, negative = net receipt): -119.361045
    LP pool inflow (USDG): -199.561045
    treasury inflow (USDG): 80.000000
    keeper execution bounty (USDG): 0.200000
Suite result: ok. 2 passed; 0 failed; 0 skipped;
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

### `packages/perps/test/perps/poc/CloseFeeTreasuryInsight.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PocBase} from "./PocBase.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice Insight PoC: the 4 bps execution fee reaches the protocol treasury on an open whose VPI rebate covers it
///         (topped up from free pool cash), but not on a close whose VPI rebate covers it (it stays in the pool).
/// @dev Close path: CfdEnginePlanLib._planCloseActionSettlement caps `delta.executionFeeAssets` to the withheld or
///      collected action charge (L1066-1068); when the close nets to a rebate that cap is zero, so the top-up computed
///      from it (L984-987) is zero and CfdEngineSettlementSidecar (L626-628) credits nothing. Open path:
///      CfdEngineSettlementSidecar.executeOpen calls _settleProtocolFeeTopUp with the uncapped fee (L498).
///      Pool depth is the fixture's 1,002,000 USDG; release risk parameters (vpiFactor 0.01) from PocBase.
contract CloseFeeTreasuryInsight is PocBase {

    uint256 internal constant MARK = 1e8;
    uint256 internal constant SIZE = 200_000e18; // $200k notional at 1.00
    uint256 internal constant MARGIN = 5_000e6;
    uint256 internal constant FEE = 80e6; // 4 bps of $200k

    address internal alice = makeAddr("alice");
    address internal bob = makeAddr("bob");

    function setUp() public override {
        super.setUp();
        _fundTrader(alice, 50_000e6);
        _fundTrader(bob, 50_000e6);
    }

    /// A skew-increasing open pays VPI + fee (fee credited to treasury). Closing the same position at the same price
    /// is skew-reducing: its VPI rebate exceeds the fee, the close nets to a rebate, and the treasury receives nothing.
    function test_insight_closeRebateKeepsExecutionFeeInPool() public {
        emit log("STEP 1: Alice opens a $200k LONG on an empty book (skew-increasing: pays VPI + 4 bps fee)");
        Flows memory open = _measure(alice, CfdTypes.Side.LONG, false);
        _logFlows(open);

        emit log("STEP 2: Alice closes the same position at the same price (skew-reducing: VPI rebate > fee)");
        Flows memory close = _measure(alice, CfdTypes.Side.LONG, true);
        _logFlows(close);

        int256 roundTrip = open.traderCost + close.traderCost;
        emit log("RESULT: round trip at an unchanged price (VPI nets out)");
        emit log_named_decimal_int("  trader round-trip cost (USDG)", roundTrip, 6);
        emit log_named_decimal_int("  treasury total (USDG)", open.treasuryInflow + close.treasuryInflow, 6);
        emit log_named_decimal_int("  LP pool total (USDG)", open.poolInflow + close.poolInflow, 6);
        emit log_named_decimal_int("  keeper execution bounties total (USDG)", open.keeperInflow + close.keeperInflow, 6);

        assertEq(open.treasuryInflow, int256(FEE), "open: the 4 bps fee is credited to the treasury");
        assertLt(close.traderCost, 0, "close nets to a rebate (VPI rebate larger than fee + carry)");
        assertEq(close.treasuryInflow, 0, "close: no execution fee reaches the treasury");
        assertGe(roundTrip, int256(2 * FEE), "the trader still bore both 4 bps fees over the round trip");
        assertEq(
            roundTrip,
            open.poolInflow + close.poolInflow + open.treasuryInflow + close.treasuryInflow + open.keeperInflow
                + close.keeperInflow,
            "every unit is accounted for (pool + treasury + keeper)"
        );
    }

    /// Control for the open path: a skew-reducing open whose VPI rebate covers the fee also nets to a rebate, yet the
    /// fee is topped up to the treasury from free pool cash.
    function test_insight_openRebateStillCreditsExecutionFee() public {
        emit log("STEP 1: Alice opens a $200k LONG (creates long skew)");
        _measure(alice, CfdTypes.Side.LONG, false);

        emit log("STEP 2: Bob opens a $200k SHORT (skew-reducing open: VPI rebate > fee)");
        Flows memory open = _measure(bob, CfdTypes.Side.SHORT, false);
        _logFlows(open);

        assertLt(open.traderCost, 0, "open nets to a rebate");
        assertEq(open.treasuryInflow, int256(FEE), "open path tops the fee up to the treasury from pool cash");
    }

    struct Flows {
        int256 traderCost;
        int256 poolInflow;
        int256 treasuryInflow;
        int256 keeperInflow;
    }

    function _measure(
        address trader,
        CfdTypes.Side side,
        bool isClose
    ) internal returns (Flows memory f) {
        address treasury = engine.protocolTreasury();
        int256 traderBefore = int256(clearinghouse.balanceAssets(trader));
        int256 poolBefore = int256(pool.totalAssets());
        int256 treasuryBefore = int256(clearinghouse.balanceAssets(treasury));
        int256 keeperBefore = _keeperWealth();
        uint64 orderId = _commit(trader, side, SIZE, isClose ? 0 : MARGIN, isClose);
        _execute(orderId, MARK);
        assertEq(_positionSize(trader), isClose ? 0 : SIZE, "order filled through the real Router");
        f.traderCost = traderBefore - int256(clearinghouse.balanceAssets(trader));
        f.poolInflow = int256(pool.totalAssets()) - poolBefore;
        f.treasuryInflow = int256(clearinghouse.balanceAssets(treasury)) - treasuryBefore;
        f.keeperInflow = _keeperWealth() - keeperBefore;
    }

    function _keeperWealth() internal view returns (int256) {
        return int256(clearinghouse.balanceAssets(keeper) + settlementToken.balanceOf(keeper));
    }

    function _logFlows(
        Flows memory f
    ) internal {
        emit log_named_decimal_int("  trader cost (USDG, negative = net receipt)", f.traderCost, 6);
        emit log_named_decimal_int("  LP pool inflow (USDG)", f.poolInflow, 6);
        emit log_named_decimal_int("  treasury inflow (USDG)", f.treasuryInflow, 6);
        emit log_named_decimal_int("  keeper execution bounty (USDG)", f.keeperInflow, 6);
    }

}
```
