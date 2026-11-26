# \#99983 [SC-Insight] Self-liquidation makes the liquidation preview understate the final account settlement by the keeper bounty

**Submitted on Nov 16th 2026 at 00:14:59 UTC by @Nescient for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99983
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# Description

## Brief/Intro

`LiquidationPreview.settlementRetainedAssets` is documented as the **total projected account settlement after liquidation debits and any immediate payout**. `ACCOUNTING_SPEC.md` is more explicit: it calls this field the projected settlement balance and an account-balance projection.

The planner always subtracts the complete liquidation charge, including the keeper share, when it calculates this field. That is exact when an unrelated keeper liquidates the account. It is not exact when the trader performs its own permissionless liquidation.

The public Router forwards unrestricted `msg.sender` as the keeper, and no check requires the keeper to differ from the liquidated account. During live Clearinghouse settlement, the full charge is first debited from the account and the keeper share is then credited to the keeper. When both addresses are the same, the account receives that share back. The final account balance is therefore:

```text
preview.settlementRetainedAssets + preview.keeperBountyAssets
```

The PoC runs at the release parameters in `DEPLOYMENT.md` and uses only public protocol entry points. From one reachable snapshot, it compares an external-keeper liquidation with self-liquidation. The Router-aware preview promises a `20.000000` USDG final account balance and separately reports a `5.000000` USDG keeper bounty. The external-keeper branch ends at exactly `20.000000` USDG; the self-liquidation branch ends at `25.000000` USDG. The preview error is therefore exactly the bounty.

## Vulnerability Details

### The preview subtracts the gross liquidation charge

`CfdEnginePlanLib` computes the projected account settlement by subtracting the entire `liquidationChargeAssets`:

```solidity
uint256 settlementAfterDebitsAssets = snap.accountBuckets.settlementBalanceAssets
    - delta.pricePnlPledgeConsumedAssets - delta.actionChargeCollectedAssets - delta.liquidationChargeAssets;
delta.settlementRetainedAssets =
    settlementAfterDebitsAssets + (delta.freshPayoutIsImmediate ? delta.freshTraderPayoutAssets : 0);
```

`CfdEngineLens` exposes that amount unchanged through `previewLiquidation`, `previewRouterLiquidation`, and `simulateLiquidation`:

```solidity
preview.keeperBountyAssets = delta.keeperBountyAssets;
// ...
preview.settlementRetainedAssets = delta.settlementRetainedAssets;
```

None of these functions receives a keeper address. They therefore cannot account for recipient aliasing.

### The public liquidation path permits the account to be its own keeper

The single-account Router path passes the original external caller as the keeper:

```solidity
host.executeLiquidationBatchItem(
    account, snapshot.price, snapshot.price, snapshot.markPrice, snapshot.publishTime, msg.sender, riskOffCutoff
);
```

There is no `msg.sender != account` validation. `CfdEngine._applyLiquidation` instead contains explicit equality-aware branches, confirming that this alias is supported rather than unreachable:

```solidity
bool syncKeeper = delta.keeperBountyAssets > 0 && keeper != delta.account && _positions[keeper].lots > 0;
// ...
if (delta.keeperBountyAssets > 0 && keeper != delta.account) {
    _checkpointBountyRecipient(keeper, delta.price, publishTime);
}
keeperBountyAssets = settlementSidecar.executeLiquidation(delta, publishTime, keeper);
```

### Live settlement credits the omitted amount back to the same account

The Clearinghouse first debits the price seizure and gross charge from the liquidated account. It then credits the keeper share:

```solidity
settlementBalances[account] -= settlementDebitAssets;
// ...
settlementBalances[execution.keeper] += execution.keeperBountyAssets;
```

When `execution.keeper == account`, that second mutation returns the keeper share to the same balance whose total the preview promised to project. The preview has no compensating addition.

## End-to-End Result

The PoC runs at the release parameters and performs the following public sequence:

1. A Junior LP supplies `5,000,000` USDG through `requestDeposit`, public `OrderRouter.settleLpEpoch`, and `claimDeposit`.
2. The trader deposits `300` USDG through the public Clearinghouse.
3. The trader commits, and a permissionless executor executes, a public V3 `10,000`-token LONG with `110` USDG margin at `1.00`.
4. The trader withdraws all but `20` USDG of free settlement, leaving the position liquidatable at `1.01`.
5. `previewRouterLiquidation(account, 1.01e8)` reports `20.000000` USDG retained settlement, a `5.000000` USDG keeper bounty, and zero immediate payout. The zero immediate payout makes the trace independent of LOW-19.
6. In the external-keeper control branch, public liquidation leaves the trader with exactly the previewed `20.000000` USDG and credits `5.000000` USDG to the external keeper.
7. After reverting only to the common snapshot, the trader publicly liquidates its own account. The trader ends with `25.000000` USDG: the previewed `20.000000` plus the `5.000000` keeper bounty.
8. Both branches leave the HousePool with the same USDG token balance. Only the recipient identity changes whether the separately reported keeper value is outside or inside the projected account.

Every equality is asserted exactly, without an approximation allowance.

## Impact Details

The demonstrated defect is an exact failure of the documented account-balance projection. For the same reachable liquidation state, the Router-aware preview promises `20.000000` USDG, while public self-liquidation leaves the projected account with `25.000000` USDG. The understated `5.000000` USDG is exactly the separately reported keeper bounty.

This report does not claim theft, insolvency, an unauthorized keeper reward, or loss of net value. The self-liquidating caller is legitimately entitled to the keeper bounty, and the control branch proves that the same value would otherwise be credited to an external keeper. Both branches produce the same HousePool token balance. The claimed impact is limited to the contest's Low category: the preview fails to deliver its promised final-account return value without losing value.

## Why This Is Not a Known Issue

- **LOW-19 is the closest known issue because it concerns the same `settlementRetainedAssets` field and overlaps the same planner/Lens functions, but its root cause and fix are different.** LOW-19 reported that an immediate trader payout was already included in `settlementRetainedAssets` and was also exposed as `immediatePayoutAssets`; ambiguous documentation could therefore make an integrator add the same payout twice and **overstate** the final balance. Its fix was to clarify that the immediate payout is already included and must not be added again.
- This report does not depend on an immediate trader payout or consumer-side addition. Its root cause is that the preview has no keeper address, always subtracts the gross liquidation charge, and therefore cannot model the live recipient alias `keeper == account`. Execution subsequently credits the keeper share back to that account, so the preview itself **understates** the final balance. Fixing G-012 requires a keeper-aware preview or narrower keeper-credit semantics; LOW-19's documentation fix cannot correct it. Conversely, adding the omitted same-account keeper credit would not resolve LOW-19's original double-counting ambiguity.
- The known liquidation findings cover payout/claim reporting, liquidation settlement, charge caps, and receipt context. None discloses recipient aliasing between the public keeper caller and the liquidated account or the resulting account-balance preview mismatch.
- The behavior is not an excluded view-contract issue. `CfdEngineLens` is specifically listed in scope.

## Recommendation

Add a keeper-aware Router preview, for example:

```solidity
previewRouterLiquidation(address account, uint256 oraclePrice, address keeper)
```

When `keeper == account`, add `keeperBountyAssets` to `settlementRetainedAssets` so the returned account-balance projection matches the live Clearinghouse balance.

If the existing Engine-level previews remain keeper-agnostic, narrow their documentation to state that `settlementRetainedAssets` excludes keeper-recipient credits and is not an exact final account balance when the keeper and account are the same. Keep the Router-facing preview caller-aware so it can fulfill the current final-account-balance promise.

Add a regression for `caller == account` that asserts exact preview/live parity.

## References

- [`ICfdEngineTypes.LiquidationPreview`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/ICfdEngineTypes.sol#L385-L430)
- [`CfdEnginePlanLib` retained-settlement calculation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1546-L1549)
- [`CfdEngineLens._liquidationPreviewFromSnapshot`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineLens.sol#L620-L648)
- [`OrderLiquidationBatchLogic` single-account keeper forwarding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L712-L729)
- [`CfdEngine._applyLiquidation`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1325-L1348)
- [`MarginClearinghouse` liquidation debit and keeper credit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1247-L1269)
- [`ACCOUNTING_SPEC.md` liquidation-preview promise](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1132-L1143)

## Proof of Concept

# Proof of Concept

The PoC uses the release risk parameters and, in `setUp`, restores the release Senior caps and Router values that the shared `BasePerpTest` fixture relaxes (`DEPLOYMENT.md:131-158`), through the owner's documented timelocked configuration paths, and uses the release 1-USDG tranche seeds. It then uses public V3 `commitOrder` and `executeOrder`, the asynchronous vault entry points, permissionless `settleLpEpoch`, public Clearinghouse deposit/withdrawal, the Router-aware preview, and public `executeLiquidation`.

Only Pyth updates and USDG are mocked, as the contest permits. There is no `vm.store`, in-scope-contract impersonation, forged in-scope balance, or legacy Router entry point. `vm.snapshotState` is used only to compare two mutually exclusive liquidation callers from the same reachable state.

Save the following file as `packages/perps/test/perps/regression/SelfLiquidationPreviewParity.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Proves that the Router-aware liquidation preview understates the liquidated account's final
///         settlement balance by exactly the keeper bounty when that account executes its own liquidation.
/// @dev Every protocol action uses a public entry point. Only Pyth and USDG are mocked, as permitted by the contest.
///      No vm.store, in-scope-contract impersonation, legacy Router entrypoint, or forged protocol balance is used.
contract SelfLiquidationPreviewParityTest is BasePerpTest {

    address private constant LIQUIDITY_PROVIDER = address(0x51F0);
    address private constant TRADER = address(0x51F1);
    address private constant OPEN_EXECUTOR = address(0x51F2);
    address private constant EXTERNAL_LIQUIDATOR = address(0x51F3);

    uint256 private constant ENTRY_PRICE = 1e8;
    uint256 private constant LIQUIDATION_PRICE = 101_000_000;
    uint256 private constant LP_ASSETS = 5_000_000e6;
    uint256 private constant TRADER_FUNDING = 300e6;
    uint256 private constant POSITION_SIZE = 10_000e18;
    uint256 private constant POSITION_MARGIN = 110e6;
    uint256 private constant FREE_SETTLEMENT_TO_RETAIN = 20e6;

    /// @dev BasePerpTest relaxes some deployment values for general testing. Restore the release values from
    ///      DEPLOYMENT.md through the owner's timelocked configuration paths, then return to the same weekday and
    ///      time as the base fixture so the market is live.
    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8_000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1_000e6;
        config.adverseConfidenceMultiplierBps = 2_500;
        _setRouterConfig(config);
        vm.warp(SETUP_TIMESTAMP + 7 days);

        assertEq(pool.maxSeniorShareBps(), 8_000, "release maxSeniorShareBps");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release maxSeniorExposureAssets");
        assertEq(router.minOpenNotionalAssets(), 1_000e6, "release minOpenNotionalAssets");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2_500, "release confidence multiplier");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10, "release confidence ratio");
        assertEq(engine.executionFeeBps(), 4, "release executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "release settlementBufferBps");
        assertEq(router.maxPendingOrders(), 5, "release maxPendingOrders");
    }

    struct ExternalLiquidationOutcome {
        uint256 accountBalance;
        uint256 keeperCredit;
        uint256 poolTokenBalance;
    }

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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// @dev Release tranche seeds are 1 USDG each (DEPLOYMENT.md); BasePerpTest defaults to 1,000 USDG.
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function test_PublicSelfLiquidationUnderstatesFinalAccountBalanceByKeeperBounty() public {
        _activateLiquidityThroughPublicRouter(juniorVault, LIQUIDITY_PROVIDER, LP_ASSETS);
        _fundTrader(TRADER, TRADER_FUNDING);
        _openThroughPublicRouter();

        uint256 freeSettlement = clearinghouse.getAccountAssetBuckets(TRADER).freeSettlementAssets;
        assertGt(freeSettlement, FREE_SETTLEMENT_TO_RETAIN, "fixture must have withdrawable settlement");
        vm.prank(TRADER);
        clearinghouse.withdraw(TRADER, freeSettlement - FREE_SETTLEMENT_TO_RETAIN);

        ICfdEngineTypes.LiquidationPreview memory preview =
        engineLens.previewRouterLiquidation(TRADER, LIQUIDATION_PRICE).liquidation;
        assertTrue(preview.liquidatable, "fixture must be liquidatable at the supplied price");
        assertEq(preview.keeperBountyAssets, 5e6, "release settings must pay a five-USDG keeper bounty");
        assertEq(preview.immediatePayoutAssets, 0, "fixture must not depend on LOW-19's immediate-payout case");
        assertGt(preview.settlementRetainedAssets, 0, "preview must promise a positive final account balance");

        uint256 commonState = vm.snapshotState();
        ExternalLiquidationOutcome memory externalOutcome = _liquidateThroughPublicRouter(EXTERNAL_LIQUIDATOR);

        assertEq(
            externalOutcome.accountBalance,
            preview.settlementRetainedAssets,
            "preview is exact when the keeper and liquidated account differ"
        );
        assertEq(
            externalOutcome.keeperCredit,
            preview.keeperBountyAssets,
            "external liquidator receives the separately reported keeper bounty"
        );

        assertTrue(vm.revertToState(commonState));
        uint256 selfLiquidatorBalanceBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolTokenBalanceBefore = settlementToken.balanceOf(address(pool));

        bytes[] memory selfLiquidationData = _mockPythUpdateData(LIQUIDATION_PRICE);
        vm.prank(TRADER);
        router.executeLiquidation(TRADER, selfLiquidationData);

        uint256 selfLiquidationFinalBalance = clearinghouse.balanceAssets(TRADER);
        uint256 selfLiquidationPoolTokenBalance = settlementToken.balanceOf(address(pool));
        uint256 previewUnderstatement = selfLiquidationFinalBalance - preview.settlementRetainedAssets;

        assertEq(
            selfLiquidationFinalBalance,
            preview.settlementRetainedAssets + preview.keeperBountyAssets,
            "self-liquidation credits the omitted keeper share back to the projected account"
        );
        assertEq(previewUnderstatement, preview.keeperBountyAssets, "the preview error is exactly the keeper bounty");
        assertEq(
            selfLiquidationFinalBalance,
            externalOutcome.accountBalance + externalOutcome.keeperCredit,
            "the same value is retained by one account instead of split between account and keeper"
        );
        assertEq(
            selfLiquidationPoolTokenBalance,
            externalOutcome.poolTokenBalance,
            "changing only the keeper identity does not change the pool-side liquidation result"
        );
        assertLt(selfLiquidationFinalBalance, selfLiquidatorBalanceBefore, "liquidation still charges the account");
        assertGt(selfLiquidationPoolTokenBalance, poolTokenBalanceBefore, "the pool still receives liquidation revenue");

        console2.log("preview settlement retained (atoms)", preview.settlementRetainedAssets);
        console2.log("reported keeper bounty (atoms)", preview.keeperBountyAssets);
        console2.log("external-keeper final account balance (atoms)", externalOutcome.accountBalance);
        console2.log("self-liquidation final account balance (atoms)", selfLiquidationFinalBalance);
        console2.log("preview understatement (atoms)", previewUnderstatement);
    }

    function _activateLiquidityThroughPublicRouter(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) private {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();

        uint256 maturity = vault.depositEpochStart(requestId);
        if (block.timestamp < maturity) {
            vm.warp(maturity);
        }
        bytes[] memory updateData = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(OPEN_EXECUTOR);
        router.settleLpEpoch(updateData);

        uint256 claimableAssets = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimableAssets, assets);
        vm.prank(lp);
        assertGt(vault.claimDeposit(requestId, claimableAssets, lp, lp), 0);
    }

    function _openThroughPublicRouter() private {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256("g012-public-open");
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = POSITION_MARGIN;
        request.targetPrice = ENTRY_PRICE;
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
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
        });

        vm.prank(TRADER);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory updateData = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(OPEN_EXECUTOR);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(result.orderId, orderId);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _liquidateThroughPublicRouter(
        address liquidator
    ) private returns (ExternalLiquidationOutcome memory outcome) {
        uint256 keeperBalanceBefore = clearinghouse.balanceAssets(liquidator);
        bytes[] memory updateData = _mockPythUpdateData(LIQUIDATION_PRICE);
        vm.prank(liquidator);
        router.executeLiquidation(TRADER, updateData);

        outcome.accountBalance = clearinghouse.balanceAssets(TRADER);
        outcome.keeperCredit = clearinghouse.balanceAssets(liquidator) - keeperBalanceBefore;
        outcome.poolTokenBalance = settlementToken.balanceOf(address(pool));
    }

}
```

Run from `packages/perps`:

```bash
FOUNDRY_PROFILE=quick forge test --root . \
  --match-path test/perps/regression/SelfLiquidationPreviewParity.t.sol \
  --match-test test_PublicSelfLiquidationUnderstatesFinalAccountBalanceByKeeperBounty -vv
```

Observed result:

```text
Ran 1 test for test/perps/regression/SelfLiquidationPreviewParity.t.sol:SelfLiquidationPreviewParityTest
[PASS] test_PublicSelfLiquidationUnderstatesFinalAccountBalanceByKeeperBounty()
Logs:
  preview settlement retained (atoms) 20000000
  reported keeper bounty (atoms) 5000000
  external-keeper final account balance (atoms) 20000000
  self-liquidation final account balance (atoms) 25000000
  preview understatement (atoms) 5000000

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

All logged values are USDG atoms with six decimals.

