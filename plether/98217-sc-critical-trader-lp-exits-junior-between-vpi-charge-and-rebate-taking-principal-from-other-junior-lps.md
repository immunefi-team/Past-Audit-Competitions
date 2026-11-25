# \#98217 [SC-Critical] Trader-LP exits Junior between VPI charge and rebate, taking principal from other Junior LPs

**Submitted on Nov 6th 2026 at 01:24:52 UTC by @anand0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98217
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Summary

A trader with mature Junior LP shares pays positive VPI on a flat-mark open, redeems at the raised NAV, then closes for the capped rebate. In the local fixture an ordinary retail LP receives **11,584.505017 USDG** on an **11,600 USDG** deposit, versus **11,599.757179 USDG** without this maneuver. The attacker ends with **15.428103 USDG** more wallet cash before external costs.

### Root cause and violated invariant

On a skew-increasing open, `CfdEngineSettlementSidecar` collects positive VPI and records the inflow as HousePool revenue ([source lines 446–465](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L465)). `HousePool.recordClaimantInflow` increments accounted cash ([lines 624–651](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L651)). LP epoch reconciliation credits the new cash to Junior principal, and redemption funding pays a queued Junior exit against that NAV ([reconciliation lines 87–99](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L87-L99), [HousePool funding lines 835–895](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L895)). The trader's own mature Junior shares can therefore claim part of the VPI charge while its position remains open.

The same trader later closes. `CloseAccountingLib` caps negative VPI by the position's prior positive VPI ([lines 103–119](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L119)), and `CfdEngineSettlementSidecar` pays the close action rebate from Pool cash ([lines 618–624](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L618-L624)). The rebate is contingent on the trader voluntarily closing, which the attacker does at the unchanged modeled mark. The preceding LP redemption is final, so other Junior holders bear the later rebate without recovering the charge value that left with the attacker's shares.

The financial invariant is that an account should not turn a flat-price, VPI-capped open/close cycle into a net cash gain by exiting its LP shares between the charge and rebate while an unrelated retail LP receives less principal at redemption. This is an ownership-allocation failure: the Pool's accounted and physical USDG still agree, and the attacker has no unpaid trader claim at the end.

### Public-user lifecycle and actual cash transfers

1. The initial pool has **39,000 USDG Senior and 61,000 USDG Junior**, including seed shares owned by accounts other than the victim. The victim acquires its Junior shares by a normal **11,600 USDG** asynchronous retail deposit, public Router epoch settlement, and share claim. The attacker owns **37,400 USDG** of mature Junior capital plus **1,000 USDG** of trader collateral; a separate Junior holder contributes the remaining nonseed Junior capital.
2. The attacker requests redemption of all its Junior shares. Shortly before the hourly maturity, it publicly commits and executes a **39,000-unit LONG** at the same neutral 1.00 mock basket mark used for settlement. The open pays **76.050000 USDG** positive VPI into HousePool.
3. At the epoch boundary, the attacker permissionlessly calls `OrderRouter.settleLpEpoch` with a fresh post-boundary mark. The pool reconciles the VPI inflow, funds the attacker redemption, and the attacker claims **37,446.133448 USDG** while the trade remains open.
4. The attacker publicly closes the position at the unchanged mock mark and withdraws free trader funds. The victim subsequently requests its own normal Junior redemption. After another public LP epoch settlement, it claims all redeemable shares for **11,584.505017 USDG**. In the otherwise identical no-trade branch, its claim transfers **11,599.757179 USDG**. The test asserts that the victim's wallet equals the received transfer and its share balance becomes zero.

`TrancheVault.requestRedeem`/`claimRedeem`, Router order commit/execute, and permissionless `OrderRouter.settleLpEpoch` are public. The PoC uses the pinned Pool, Router, Engine, Clearinghouse and Vault with permitted mock USDG/Pyth. The attacker has no owner, guardian, pauser or special keeper role.

### Measured primary result

| Actual USDG measure | No trade | Attacker exit before close | Difference |
| --- | ---: | ---: | ---: |
| Victim retail deposit | 11,600.000000 | 11,600.000000 | — |
| Victim full-share redemption transfer | 11,599.757179 | **11,584.505017** | **-15.252162** |
| Victim final USDG wallet | 11,599.757179 | **11,584.505017** | **-15.252162** |
| Attacker final USDG wallet | 38,399.504949 | **38,414.933052** | **+15.428103** |

These are actual token balances. The victim loses **15.252162 USDG versus control** and **15.494983 USDG versus deposited principal**. The attacker gains **15.428103 USDG versus control** and ends **14.933052 USDG above its 38,400 USDG funded capital**, before external costs. The victim chooses to redeem later; its NAV is already lower after the close.

With **mocked 10-bps component confidence**, accepted under the fixture's configured ceiling, the second test still passes. The victim receives **11,591.165256 USDG** instead of **11,599.757179**, a **8.591923 USDG** loss relative to no trade; the attacker ends **38,401.387752** instead of **38,399.504949**, a **1.882803 USDG** gain relative to no trade, before gas, Pyth update and Hermes costs. This sensitivity result does not establish net live profit at that confidence level.

### Matched cash and rebate control against the public scan

The [published AI scan](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1742-L1808) describes an LP/trader reducing close-time pool depth to *increase the negative close VPI rebate*. The third test in this **same runnable file** separately queues an independent new Junior deposit for the attacker's LP epoch. HousePool funds the redemption before activating that deposit ([lines 872–929](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L872-L929)), restoring close-time Pool cash.

| Measure (USDG) | Close before LP exit | LP exit before close | Difference |
| --- | ---: | ---: | ---: |
| Pool cash immediately before close | **100,076.050000** | **100,076.050000** | 0 |
| Gross closing VPI | **-75.992207** | **-75.992207** | 0 |
| Paid closing rebate | **60.392207** | **60.392207** | 0 |
| Attacker LP redemption payment | 37,409.105218 | **37,446.133448** | **+37.028230** |
| Attacker final wallet | 38,377.847402 | **38,414.875274** | **+37.027872** |
| New unrelated Junior LP NAV | 37,446.133447 | **37,409.105378** | **-37.028069** |

With equal Pool cash, closing VPI and paid rebate, attacker LP payout rises **37.028230 USDG** while another Junior holder NAV falls **37.028069 USDG**. This supports a redemption-pricing mechanism distinct from the public scan larger-rebate mechanism. The control does not prove identical internal receipt-level depth; tiny carry rounding can differ.

The [whitepaper](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L805-L813) intentionally treats positive VPI as pool income and negative VPI as a trader rebate. The [accounting specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L289-L301) intentionally prices redemptions before activating deposits. The claimed gap is their **cross-role combination**: a trader/LP profits from a flat-price cycle and a separate retail LP loses actual redemption cash even while the [documented lifetime clamp aims to prevent extractable rebate-only round trips](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1434-L1444).

### Tested ownership, scale and repeatability

A second [runnable local test](https://gist.github.com/Anand-0037/89e4104be32def894be44bc32c3067ec) passed a 13-case sweep and one five-epoch test. With a 39%-of-depth LONG, the attacker is gross-positive only at **about 60% Junior ownership** among the tested 10%, 25%, 40% and 60% cases; this is a tested condition, not a universal threshold. At about 60%, attacker wallet gain versus matched no-trade / retail victim cash loss (USDG) are: **100k pool: +14.430698 / -15.016361; 1m: +144.296534 / -150.149771; 10m: +1,442.954898 / -1,501.483887**. The effects scale approximately with pool size in these fixtures; 10m is a sensitivity case, not current TVL. Five self-funded attacker cycles with a fresh retail depositor each show **+72.336480 / -75.270062 USDG** as sums of matched per-epoch comparisons, not one global five-epoch no-trade trajectory. At 100k and 10-bps mocked confidence, gross attacker gain is only **0.758025 USDG**.

### Configuration, costs and correction

The fixture uses the pinned [release settings](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L125-L155) and permitted USDG/Pyth mocks. Gas, Pyth/Hermes charges, price movement and live liquidity are unmeasured; net live profit is not claimed. Reserve refundable positive VPI from withdrawable Junior NAV and cash until the position closes, without double-counting trader claims.

## Link to Proof of Concept

https://gist.github.com/Anand-0037/89e4104be32def894be44bc32c3067ec

## Proof of Concept

Pinned competition source: a82d4fefb02768e5b3bddcc97dee371eee24fea8. Save this complete file as packages/perps/test/perps/experiments/PoCVpiPortalMinimal.t.sol. Run from repository root:

Additional 3/3 control suite and full original source (secret Gist): https://gist.github.com/Anand-0037/89e4104be32def894be44bc32c3067ec

The same secret Gist also contains `PoCVpiScalingSweep.t.sol` (SHA-256 `db7521ee38da482d6ee2ac8291a9ac92634daa361d1c4bd19424dc338d0bb136`). Copy it to the same `experiments/` directory and run `forge test --root packages/perps --match-path test/perps/experiments/PoCVpiScalingSweep.t.sol -vv` to reproduce the 13/13 ownership/size sweep and five-epoch test.

```sh
forge test --root packages/perps --match-path test/perps/experiments/PoCVpiPortalMinimal.t.sol -vv
```

Test source SHA-256: 8ad516f146d80bc50b6d725fe109e4e4d19e37d9c4a9463da4becf56c9072a01. Fresh 6 Nov 2026 result: 1 passed, 0 failed. Victim normal cash 11,599.757179 USDG; attacked cash 11,584.505017 USDG. Attacker normal wallet 38,399.504949 USDG; attacked wallet 38,414.933052 USDG. These are local results with the repository test fixture; no deployed transaction or net live profit is claimed.

Reproduction steps: (1) The victim makes and claims an ordinary 11,600 USDG Junior deposit. (2) The attacker queues a Junior redemption, opens a 39,000-unit LONG at the neutral mark and pays positive VPI. (3) A public LP epoch settlement funds the attacker's redemption while the position is open. (4) The attacker closes at the same mark, withdraws trader cash, and the victim later redeems all shares. The test reverts to the same starting snapshot for a no-trade control and asserts actual wallet transfers and share burning. No privileged role is given to either actor.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

/// @notice Standalone, local proof on the competition's pinned production contracts.
contract PoCVpiPortalMinimalTest is HousePoolAsyncTestBase {
    address private constant ATTACKER = address(0xA771);
    address private constant VICTIM = address(0xB0B1);
    address private constant OTHER_JUNIOR = address(0xB0B3);
    address private constant SENIOR = address(0x5E510);
    uint256 private constant MARK = 1e8;

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _juniorSeedReceiver() internal pure override returns (address) { return OTHER_JUNIOR; }
    function _seniorSeedReceiver() internal pure override returns (address) { return SENIOR; }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18, maxSkewRatio: 0.4e18, maintMarginBps: 10,
            initMarginBps: 20, fadMarginBps: 300, baseCarryBps: 500,
            minBountyAssets: 1e6, bountyBps: 10, keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(cfg);
        _setSeniorCapacity(40_000_000e6, 8000);
        vm.warp(block.timestamp + 24 hours);
        _fundJunior(OTHER_JUNIOR, 11_000e6);
        _fundJunior(ATTACKER, 37_400e6);
        _fundSenior(SENIOR, 38_000e6);
        uint256 depositId = _requestAsyncDeposit(juniorVault, VICTIM, 11_600e6);
        vm.warp(juniorVault.depositEpochStart(depositId));
        bytes[] memory depositPrice = _mockPythUpdateData(MARK);
        vm.prank(VICTIM);
        router.settleLpEpoch(depositPrice);
        _claimAsyncDeposit(juniorVault, depositId, VICTIM);
        _fundTrader(ATTACKER, 1_000e6);
        _finishAsyncCooldown(juniorVault, ATTACKER);
        assertEq(pool.totalAssets(), 100_000e6);
    }

    function test_RetailPrincipalTransferredByTraderLpExitBetweenVpiChargeAndRebate() public {
        // The victim has retail-minted shares, no seed shares or remaining USDG.
        uint256 shares = juniorVault.balanceOf(VICTIM);
        assertGt(shares, 0);
        assertGt(juniorVault.balanceOf(OTHER_JUNIOR), juniorVault.seedShareFloor());
        assertEq(settlementToken.balanceOf(VICTIM), 0);
        uint256 snapshot = vm.snapshotState();

        (uint256 normalAttackerCash, uint256 normalVictimCash) = _run(false);
        assertTrue(vm.revertToState(snapshot));
        (uint256 attackedAttackerCash, uint256 attackedVictimCash) = _run(true);

        emit log_named_uint("victim_deposit_usdg", 11_600e6);
        emit log_named_uint("victim_no_trade_cash", normalVictimCash);
        emit log_named_uint("victim_attacked_cash", attackedVictimCash);
        emit log_named_uint("attacker_no_trade_cash", normalAttackerCash);
        emit log_named_uint("attacker_attacked_cash", attackedAttackerCash);

        assertEq(normalVictimCash, 11_599_757_179);
        assertEq(attackedVictimCash, 11_584_505_017);
        assertEq(normalAttackerCash, 38_399_504_949);
        assertEq(attackedAttackerCash, 38_414_933_052);
        assertLt(attackedVictimCash, 11_600e6, "victim loses deposited principal");
        assertLt(attackedVictimCash, normalVictimCash, "victim receives less actual USDG");
        assertGt(attackedAttackerCash, normalAttackerCash, "attacker receives more actual USDG");
        assertEq(juniorVault.balanceOf(VICTIM), 0, "victim full shares burned");
        assertEq(pool.totalAssets(), settlementToken.balanceOf(address(pool)));
        assertEq(engine.traderClaimBalanceAssets(ATTACKER), 0);
    }

    function _run(bool attack) private returns (uint256 attackerCash, uint256 victimCash) {
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);
        uint256 attackerRequest = _requestAsyncRedeem(juniorVault, ATTACKER, attackerShares);
        uint256 maturity = pool.lpEpochStart(attackerRequest);
        vm.warp(maturity - 15);
        if (attack) assertGt(_trade(false), 0, "opening VPI charged to trader");
        vm.warp(maturity);
        bytes[] memory attackerLpPrice = _mockPythUpdateData(MARK);
        vm.prank(ATTACKER);
        router.settleLpEpoch(attackerLpPrice);
        assertEq(juniorVault.claimableRedeemRequest(attackerRequest, ATTACKER), attackerShares);
        _claimAsyncRedeem(juniorVault, attackerRequest, ATTACKER);
        if (attack) assertLt(_trade(true), 0, "closing VPI rebated to trader");
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, free);
        attackerCash = settlementToken.balanceOf(ATTACKER);

        uint256 victimShares = juniorVault.balanceOf(VICTIM);
        uint256 victimRequest = _requestAsyncRedeem(juniorVault, VICTIM, victimShares);
        vm.warp(pool.lpEpochStart(victimRequest));
        bytes[] memory victimLpPrice = _mockPythUpdateData(MARK);
        vm.prank(VICTIM);
        router.settleLpEpoch(victimLpPrice);
        assertEq(juniorVault.claimableRedeemRequest(victimRequest, VICTIM), victimShares);
        (, victimCash) = _claimAsyncRedeem(juniorVault, victimRequest, VICTIM);
        assertEq(settlementToken.balanceOf(VICTIM), victimCash);
    }

    function _trade(bool close) private returns (int256 vpi) {
        OrderV3Types.OrderRequest memory r;
        r.clientOrderId = keccak256(abi.encode("vpi-lp-exit", router.nextCommitId()));
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = 39_000e18;
        r.marginDelta = close ? 0 : 300e6;
        r.isClose = close;
        r.targetPrice = MARK;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + 60), executionWindowSeconds: 60,
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
        vm.prank(ATTACKER);
        uint64 id = router.commitOrder(r);
        bytes[] memory executionPrice = _mockPythUpdateData(MARK);
        _startRecordingLogs();
        vm.prank(ATTACKER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, executionPrice);
        _takeRecordedLogs();
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        vpi = _verifiedReceipt(router.lifecycleBook(), id).economics.vpiAssets;
    }
}

```

