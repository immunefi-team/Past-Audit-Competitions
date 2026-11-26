# \#99709 [SC-Critical] # Security Best Practices Insight: Inconsistent pricing when assigning `unassignedAssets` dilutes existing LPs' recovery entitlement

**Submitted on Nov 14th 2026 at 21:44:51 UTC by @dikanevn for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99709
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**

## Description

## Report Information

- **Protocol / Repo / Commit:** Plether / `https://github.com/immunefi-team/audit-comp-plether` (`a82d4fefb02768e5b3bddcc97dee371eee24fea8`)
- **Assets (In Scope):**
  - `packages/perps/src/HousePool.sol` (lines 660–704)
  - `packages/perps/src/TrancheVault.sol` (lines 1392–1396, 1853–1859)
  - `packages/perps/src/libraries/HousePoolTrancheGateLib.sol` (lines 28–41)
- **Severity:** `Insight`
- **Impact:** `Security Best practices`
- **Target Function:** `HousePool.assignUnassignedAssets(bool toSenior, address receiver)`
- **Program:** [Plether Audit Competition — Information](https://immunefi.com/audit-competition/plether/information/) / [Scope](https://immunefi.com/audit-competition/plether/scope/)

## Summary

When a tranche has zero principal but existing LP shares remain outstanding, the ordinary deposit pricing function, `estimateDepositShares`, refuses to quote new shares. In contrast, `quoteBootstrapDeposit`, used by `assignUnassignedAssets`, continues pricing with a virtual denominator of one base unit of the asset. The resulting share issuance transfers almost the entire entitlement to future tranche revenue—the recovery entitlement preserved for existing holders by the ordinary deposit path—to `receiver`.

This occurs when allocating the quarantined `unassignedAssets` balance. That balance blocks new deposits, while the Senior capacity limit prevents assigning it to Senior in the PoC state. Assigning it to Junior clears the deposit gate at the cost of severely diluting existing LPs. The sections below describe the mechanism, numerical results, and a separate way to restore principal without minting shares.

## Preconditions and Operational Scenario

1. A trading payout wipes out Junior principal (`juniorPrincipal == 0`), while LP shares remain outstanding (`totalSupply > 0`).
2. The owner calls `CfdEngine.recapitalizePool`. The funds restore Senior principal to its high-water mark (HWM), and any excess enters `unassignedAssets`. According to `ACCOUNTING_SPEC.md:357`, this bucket also receives revenue with no eligible share owner.
3. While `unassignedAssets > 0`, `HousePoolTrancheGateLib.sol:28–41` blocks new deposits (`trancheDepositsAllowed == false`). The balance is already accounted for in the pool's state and cannot be swept as unaccounted excess.
4. `assignUnassignedAssets` is the designated allocation function. With `toSenior = true`, `_requireSeniorDepositCapacity` rejects the allocation in the PoC state: the wiped Junior tranche cannot support the required tranche ratio. With `toSenior = false`, the function invokes Junior's bootstrap pricing function, described below.

## Root Cause: Inconsistent Pricing Rules at Zero NAV

```solidity
// TrancheVault.sol:1392-1396
function quoteBootstrapDeposit(uint256 assets) external view returns (uint256) {
    return _convertToSharesUsingAssets(assets, _depositPricingAssets(), Math.Rounding.Floor);
}

// TrancheVault.sol:1853-1859
function _convertToSharesUsingAssets(uint256 assets, uint256 totalAssets_, Math.Rounding rounding)
    internal view returns (uint256)
{
    return Math.mulDiv(assets, accruedTotalSupply() + VIRTUAL_SHARES, totalAssets_ + 1, rounding);
}
```

The formula is `shares = assets * (accruedTotalSupply + 1000) / (totalAssets_ + 1)`.

In this state, `totalAssets_ == 0`, so the denominator is one base unit of USDG (`0.000001 USDG`). For each base unit of assigned assets, the receiver receives `accruedTotalSupply + 1000` base units of shares. The virtual offset in the conversion formula therefore determines the entry price into a tranche that still has existing holders.

The ordinary deposit pricing function explicitly refuses this valuation:

```solidity
// TrancheVault.sol:1361-1369
function estimateDepositShares(uint256 assets) public view returns (uint256) {
    uint256 pricingAssets = _depositPricingAssets();
    uint256 pricingSupply = accruedTotalSupply();
    if (pricingAssets == 0 && pricingSupply != 0) {
        return 0;
    }
    // ...
}
```

Ordinary deposits into a terminally wiped tranche also revert with `TrancheVault__TerminallyWiped`; `_isTerminallyWiped()` checks `totalSupply() > 0 && totalAssets() == 0`. The bootstrap path lacks the corresponding protection.

## Economic Impact Demonstrated by the PoC

BOB and CHARLIE initially deposit 10,000 USDG each into Junior. After the principal is wiped out and the owner recapitalizes the pool with 2,000 USDG, the balance available for assignment is 1,817.158916 USDG.

- Shares minted to `TREASURY`: 36,369,216,160,598,375,750,284 share base units.
- `totalSupply` after assignment: 36,369,216,180,612,704,622,983 share base units.
- BOB / CHARLIE balances: 10,006,664,431,847 / 10,006,664,440,852 share base units; unchanged by assignment.
- BOB / CHARLIE ownership using `(balance * 10000) / totalSupply`: 0 / 0 BPS after rounding down.
- `TREASURY` ownership using the same formula: 9999 BPS after rounding down.
- `convertToAssets` for each original LP after 50,000 USDG of revenue is admitted: 14 USDG base units (`0.000014 USDG`).
- Actual payout to BOB in the illustrated redemption epoch: 8 USDG base units (`0.000008 USDG`).
- Actual payout to `TREASURY` in the same epoch: 31,815.082636 USDG.

Each original LP's relative ownership rounds down to 0 BPS; their absolute share balances remain unchanged. `convertToAssets` gives the calculated position value, while the actual payout depends on the conditions of the individual asynchronous redemption epoch.

A control test recapitalizes the pool by exactly the Senior shortfall, with no assignment and no new share issuance. It demonstrates that existing shares retain their recovery entitlement: after the same revenue is admitted, BOB and CHARLIE each have a calculated position value of approximately 24,998.75 USDG. If BOB receives the bootstrap shares instead, almost all recovered value accrues to him, while CHARLIE's position is again valued at just 14 USDG base units.

## Specification and Administrative Asset Assignment

`ACCOUNTING_SPEC.md:325–327` establishes the following principles:

> a wiped tranche cannot be silently revived by an ordinary asynchronous deposit

> seeded ownership continuity is preferred over governance re-assignment.

These principles describe ownership continuity for ordinary asynchronous deposits. Recapitalization is described separately: funds cover the Senior shortfall or enter `unassignedAssets` without minting shares.

For administrative assignment of `unassignedAssets`, the NatSpec for `assignUnassignedAssets` (`HousePool.sol:653–659`) explicitly provides for minting shares to a designated receiver:

```text
@notice Explicitly bootstraps quarantined LP assets into a tranche by minting matching shares.
@param receiver Account receiving bootstrap tranche shares
```

The trusted owner calls this function and selects `receiver`. At zero NAV with nonzero share supply, the virtual offset determines the size of the bootstrap issuance. In the PoC state, this concentrates almost all relative tranche ownership and future revenue in the receiver. The current assignment path does not provide a separate mode for restoring Junior principal while preserving the existing ownership distribution.

## Recommended Mitigation

Separate share issuance to a receiver from principal restoration for existing holders:

1. **Align the pricing guards.** In `quoteBootstrapDeposit`, check `pricingAssets == 0 && pricingSupply != 0` and revert with `TrancheVault__TerminallyWiped`. When `pricingSupply` is zero, this check does not prevent initial seeding of an empty tranche through `initializeSeedPosition`.
2. **Add an explicit restoration function that does not mint shares**, such as `reinstateTranchePrincipal`. It should move `unassignedAssets` into the selected tranche's principal, subject to that tranche's constraints. It should not accept a `receiver`: the restored value belongs to existing holders in proportion to their shares, including the seed floor. Allocating the balance clears the corresponding deposit gate.

The existing `assignUnassignedAssets` function is intended to mint shares to `receiver`. Principal restoration without share issuance should be implemented as a separate function with explicit ownership-continuity semantics. The bootstrap pricing guard prevents issuance into a wiped tranche with existing holders; the separate restoration function allows the quarantined balance to be allocated in that state.


## Proof of Concept

## Proof of Concept (Foundry)

`AssignUnassignedZeroNavBootstrapPoC_v6.t.sol` uses the actual Plether contracts (`HousePool`, `TrancheVault`, `CfdEngine`, and `OrderRouter`) with the release parameters from `DEPLOYMENT.md`. Junior is wiped out through a Router trade, without `vm.store` or `stdstore`. Future revenue is modeled by increasing the pool balance and calling `accountExcess`.

The suite contains three tests:

- `test_Control_JuniorRevenueAccruesToExistingLpsProportionally` — control case demonstrating recovery of existing share value.
- `test_PoC_AssignToTreasuryDilutesAllExistingLpsToZero` — assignment to the treasury, checking pricing, ownership, and actual payouts.
- `test_PoC_AssignToExistingLpExpropriatesOtherLps` — assignment to one of the existing LPs.

### Reproduction Instructions from Scratch

#### 1. Clone the Repository and Check Out the In-Scope Commit

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

#### 2. Create the PoC Test File

```bash
mkdir -p packages/perps/test/perps/poc
cat << 'EOF' > packages/perps/test/perps/poc/AssignUnassignedZeroNavBootstrapPoC_v6.t.sol
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice Architecture & Security Best Practice PoC.
/// @dev Demonstrates that when unassignedAssets > 0 and juniorPrincipal == 0,
///      HousePool.assignUnassignedAssets(false, receiver) invokes quoteBootstrapDeposit
///      which uses totalAssets + 1 = 1 atom as denominator, minting trillions of shares
///      and severely diluting existing LPs (BOB and CHARLIE).
contract AssignUnassignedZeroNavBootstrapPoC_v6 is BasePerpTest {
    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant CHARLIE = address(0xC4A2);
    address internal constant TREASURY = address(0x78EA5);
    address internal constant TRADER = address(0x7AAD);
    address internal constant KEEPER = address(0x4EE9);

    uint256 internal constant JUNIOR_DEPOSIT_BOB = 10_000e6;
    uint256 internal constant JUNIOR_DEPOSIT_CHARLIE = 10_000e6;
    uint256 internal constant SENIOR_DEPOSIT = 80_000e6;
    uint256 internal constant RECAP = 2_000e6;
    uint256 internal constant REVENUE = 50_000e6;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// @dev Release seeds are 1 USDG per tranche (DEPLOYMENT.md).
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    /// @dev DEPLOYMENT.md initial release parameters.
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

    function _reconcileAsVault() internal {
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function _ownerRecap(uint256 amount) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
        _reconcileAsVault();
    }

    /// @dev Models future protocol trading revenue (carry/fees/liquidations) via accountExcess.
    function _admitRevenue(uint256 amount) internal {
        settlementToken.mint(address(pool), amount);
        pool.accountExcess();
        _reconcileAsVault();
    }

    function _routerOpen(CfdTypes.Side side, uint256 size, uint256 margin, uint256 price) internal {
        vm.prank(TRADER);
        uint64 id = router.commitOrder(side, size, margin, price, false);
        vm.prank(KEEPER);
        router.executeOrder(id, _mockPythUpdateData(price));
    }

    function _routerClose(CfdTypes.Side side, uint256 size, uint256 price) internal {
        vm.prank(TRADER);
        uint64 id = router.commitOrder(side, size, 0, price, true);
        vm.prank(KEEPER);
        router.executeOrder(id, _mockPythUpdateData(price));
    }

    /// @dev Produces a genuine Junior wipe through the production Router entrypoints.
    ///      Both BOB and CHARLIE deposit 10,000 USDG each (50/50 split).
    function _wipeJuniorWithRealRouterTrade() internal {
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(BOB, JUNIOR_DEPOSIT_BOB);
        _fundJunior(CHARLIE, JUNIOR_DEPOSIT_CHARLIE);
        _fundSenior(ALICE, SENIOR_DEPOSIT);

        uint256 initialJuniorPrincipal = pool.juniorPrincipal();
        emit log_named_uint("01. junior principal after LP deposits", initialJuniorPrincipal);

        uint256 sizeTokens = (initialJuniorPrincipal / 1e6) * 101 / 100;
        _fundTrader(TRADER, 2_500e6);
        _routerOpen(CfdTypes.Side.SHORT, sizeTokens * 1e18, 2_000e6, 1e8);
        _routerClose(CfdTypes.Side.SHORT, sizeTokens * 1e18, 2e8);
        _reconcileAsVault();

        assertEq(pool.juniorPrincipal(), 0, "junior principal wiped by a real Router trade");
        assertGt(juniorVault.totalSupply(), 0, "existing junior shares survive the wipe");
        assertGt(juniorVault.balanceOf(BOB), 0, "BOB still owns junior shares");
        assertGt(juniorVault.balanceOf(CHARLIE), 0, "CHARLIE still owns junior shares");
        assertLt(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "senior impaired by the same trade");
        assertEq(pool.totalAssets(), pool.seniorPrincipal(), "the loss is real cash leaving the pool");

        emit log_named_uint("02. junior principal after loss", pool.juniorPrincipal());
        emit log_named_uint("03. junior supply after loss", juniorVault.totalSupply());
        emit log_named_uint("04. BOB shares after loss", juniorVault.balanceOf(BOB));
        emit log_named_uint("05. CHARLIE shares after loss", juniorVault.balanceOf(CHARLIE));
        emit log_named_uint("06. senior principal after loss", pool.seniorPrincipal());
        emit log_named_uint("07. senior HWM after loss", pool.seniorHighWaterMark());
        emit log_named_uint("08. senior HWM gap to restore", pool.seniorHighWaterMark() - pool.seniorPrincipal());
    }

    /// @dev Differential control: identical loss, recap covering Senior gap exactly,
    ///      and identical revenue. Both BOB and CHARLIE receive their fair 50/50 share of revenue.
    function test_Control_JuniorRevenueAccruesToExistingLpsProportionally() public {
        _wipeJuniorWithRealRouterTrade();
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        _ownerRecap(gap);
        assertEq(pool.unassignedAssets(), 0, "gap-only recap leaves no unassigned bucket");

        _admitRevenue(REVENUE);

        uint256 bobValue = juniorVault.convertToAssets(juniorVault.balanceOf(BOB));
        uint256 charlieValue = juniorVault.convertToAssets(juniorVault.balanceOf(CHARLIE));
        emit log_named_uint("C1. junior principal after revenue", pool.juniorPrincipal());
        emit log_named_uint("C2. BOB redeemable value after revenue", bobValue);
        emit log_named_uint("C3. CHARLIE redeemable value after revenue", charlieValue);
        assertGt(bobValue, 24_500e6, "BOB owns ~50% of the recognized Junior revenue");
        assertGt(charlieValue, 24_500e6, "CHARLIE owns ~50% of the recognized Junior revenue");
    }

    /// @dev Main PoC: unassignedAssets exists after recapitalization (e.g. slight overfunding or async gap).
    ///      The deposit gate is blocked. The owner calls assignUnassignedAssets(false, TREASURY) to clear
    ///      the bucket and revive Junior.
    ///      Both BOB and CHARLIE are diluted from ~5000 BPS each to 0 BPS (rounding).
    function test_PoC_AssignToTreasuryDilutesAllExistingLpsToZero() public {
        _wipeJuniorWithRealRouterTrade();

        // Ordinary deposit refuses at zero NAV
        assertEq(juniorVault.estimateDepositShares(1_000e6), 0, "ordinary deposit path refuses to price at zero NAV");
        uint256 bootstrapQuote = juniorVault.quoteBootstrapDeposit(1_000e6);
        emit log_named_uint("10. estimateDepositShares(1000 USDG) at zero NAV", juniorVault.estimateDepositShares(1_000e6));
        emit log_named_uint("11. quoteBootstrapDeposit(1000 USDG) at zero NAV", bootstrapQuote);
        assertGt(bootstrapQuote, 1e22, "bootstrap path prices deposit using totalAssets + 1 = 1 atom");

        // Recapitalization creates unassignedAssets (Senior restored, excess quarantined)
        _ownerRecap(RECAP);
        uint256 unassigned = pool.unassignedAssets();
        emit log_named_uint("12. senior principal after recap", pool.seniorPrincipal());
        emit log_named_uint("13. unassignedAssets after recap", unassigned);
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "senior fully restored");
        assertGt(unassigned, 1_000e6, "recap overflow is quarantined as unassignedAssets");
        assertFalse(pool.canAcceptTrancheDeposits(false), "unassigned bucket blocks new Junior deposits");

        // Owner clears bucket to unfreeze deposits and revive Junior
        pool.assignUnassignedAssets(false, TREASURY);
        assertEq(pool.unassignedAssets(), 0, "bucket cleared");
        assertTrue(pool.canAcceptTrancheDeposits(false), "deposit gate reopens");

        uint256 bobShareBps = (juniorVault.balanceOf(BOB) * 10_000) / juniorVault.totalSupply();
        uint256 charlieShareBps = (juniorVault.balanceOf(CHARLIE) * 10_000) / juniorVault.totalSupply();
        uint256 treasuryShareBps = (juniorVault.balanceOf(TREASURY) * 10_000) / juniorVault.totalSupply();
        emit log_named_uint("14. junior principal after assign", pool.juniorPrincipal());
        emit log_named_uint("15. junior supply after assign", juniorVault.totalSupply());
        emit log_named_uint("16. BOB shares after assign", juniorVault.balanceOf(BOB));
        emit log_named_uint("17. CHARLIE shares after assign", juniorVault.balanceOf(CHARLIE));
        emit log_named_uint("18. treasury shares after assign", juniorVault.balanceOf(TREASURY));
        emit log_named_uint("19. BOB junior share in BPS", bobShareBps);
        emit log_named_uint("20. CHARLIE junior share in BPS", charlieShareBps);
        emit log_named_uint("21. treasury junior share in BPS", treasuryShareBps);

        // Both original LPs are rounded down to 0 BPS
        assertEq(bobShareBps, 0, "BOB share rounds down to zero BPS");
        assertEq(charlieShareBps, 0, "CHARLIE share rounds down to zero BPS");
        assertEq(treasuryShareBps, 9999, "treasury captures 99.9999999% of the tranche");

        // Future revenue accrues almost entirely to treasury
        _admitRevenue(REVENUE);

        uint256 bobValue = juniorVault.convertToAssets(juniorVault.balanceOf(BOB));
        uint256 charlieValue = juniorVault.convertToAssets(juniorVault.balanceOf(CHARLIE));
        uint256 treasuryValue = juniorVault.convertToAssets(juniorVault.balanceOf(TREASURY));
        emit log_named_uint("22. junior principal after revenue", pool.juniorPrincipal());
        emit log_named_uint("23. BOB redeemable value after revenue", bobValue);
        emit log_named_uint("24. CHARLIE redeemable value after revenue", charlieValue);
        emit log_named_uint("25. treasury redeemable value after revenue", treasuryValue);
        assertLt(bobValue, 1e6, "BOB keeps less than 1 USDG");
        assertLt(charlieValue, 1e6, "CHARLIE keeps less than 1 USDG");
        assertGt(treasuryValue, 50_000e6, "treasury captures entire revenue stream");

        // Cash payout through asynchronous redemption epoch
        uint256 bobSharesToRedeem = juniorVault.balanceOf(BOB);
        uint256 treasurySharesToRedeem = juniorVault.balanceOf(TREASURY);
        vm.warp(block.timestamp + 1 hours + 1);
        vm.prank(BOB);
        uint256 bobRequest = juniorVault.requestRedeem(bobSharesToRedeem, BOB, BOB);
        vm.prank(TREASURY);
        uint256 treasuryRequest = juniorVault.requestRedeem(treasurySharesToRedeem, TREASURY, TREASURY);

        vm.warp(juniorVault.depositEpochStart(treasuryRequest));
        _settleLpEpochForTest();

        uint256 bobClaimShares = juniorVault.claimableRedeemRequest(bobRequest, BOB);
        uint256 treasuryClaimShares = juniorVault.claimableRedeemRequest(treasuryRequest, TREASURY);
        uint256 bobBefore = settlementToken.balanceOf(BOB);
        vm.prank(BOB);
        juniorVault.claimRedeem(bobRequest, bobClaimShares, BOB, BOB);
        uint256 treasuryBefore = settlementToken.balanceOf(TREASURY);
        vm.prank(TREASURY);
        juniorVault.claimRedeem(treasuryRequest, treasuryClaimShares, TREASURY, TREASURY);

        uint256 paidBob = settlementToken.balanceOf(BOB) - bobBefore;
        uint256 paidTreasury = settlementToken.balanceOf(TREASURY) - treasuryBefore;
        emit log_named_uint("26. paid to BOB in redeem epoch", paidBob);
        emit log_named_uint("27. paid to treasury in redeem epoch", paidTreasury);
        assertLt(paidBob, 100, "BOB cannot withdraw any meaningful revenue");
        assertGt(paidTreasury, 20_000e6, "treasury withdraws real cash");
    }

    /// @dev Demonstrates asymmetric dilution when directing bootstrap shares to BOB
    function test_PoC_AssignToExistingLpExpropriatesOtherLps() public {
        _wipeJuniorWithRealRouterTrade();
        _ownerRecap(RECAP);

        pool.assignUnassignedAssets(false, BOB);

        uint256 bobShareBps = (juniorVault.balanceOf(BOB) * 10_000) / juniorVault.totalSupply();
        uint256 charlieShareBps = (juniorVault.balanceOf(CHARLIE) * 10_000) / juniorVault.totalSupply();
        emit log_named_uint("30. BOB junior share in BPS when receiver=BOB", bobShareBps);
        emit log_named_uint("31. CHARLIE junior share in BPS when receiver=BOB", charlieShareBps);

        assertEq(bobShareBps, 9999, "BOB captures 99.9999999% of the tranche");
        assertEq(charlieShareBps, 0, "CHARLIE share rounds down to 0 BPS");

        // Future revenue accrues to BOB, CHARLIE gets virtually nothing
        _admitRevenue(REVENUE);

        uint256 bobValue = juniorVault.convertToAssets(juniorVault.balanceOf(BOB));
        uint256 charlieValue = juniorVault.convertToAssets(juniorVault.balanceOf(CHARLIE));
        emit log_named_uint("32. BOB redeemable value after revenue", bobValue);
        emit log_named_uint("33. CHARLIE redeemable value after revenue", charlieValue);
        assertGt(bobValue, 50_000e6, "BOB took all future revenue");
        assertLt(charlieValue, 1e6, "CHARLIE received virtually 0");
    }
}
EOF
```

#### 3. Run the Tests

```bash
cd packages/perps
forge test --match-contract AssignUnassignedZeroNavBootstrapPoC_v6 -vv
```

### Execution Logs Provided with the Original Report

```text
Ran 3 tests for test/perps/poc/AssignUnassignedZeroNavBootstrapPoC_v6.t.sol:AssignUnassignedZeroNavBootstrapPoC_v6
[PASS] test_Control_JuniorRevenueAccruesToExistingLpsProportionally() (gas: 11951227)
Logs:
  01. junior principal after LP deposits: 20000999316
  02. junior principal after loss: 0
  03. junior supply after loss: 20014328872699
  04. BOB shares after loss: 10006664431847
  05. CHARLIE shares after loss: 10006664440852
  06. senior principal after loss: 79818160005
  07. senior HWM after loss: 80001001089
  08. senior HWM gap to restore: 182841084
  C1. junior principal after revenue: 50000000000
  C2. BOB redeemable value after revenue: 24998750882
  C3. CHARLIE redeemable value after revenue: 24998750905

[PASS] test_PoC_AssignToExistingLpExpropriatesOtherLps() (gas: 12220525)
Logs:
  01. junior principal after LP deposits: 20000999316
  02. junior principal after loss: 0
  03. junior supply after loss: 20014328872699
  04. BOB shares after loss: 10006664431847
  05. CHARLIE shares after loss: 10006664440852
  06. senior principal after loss: 79818160005
  07. senior HWM after loss: 80001001089
  08. senior HWM gap to restore: 182841084
  30. BOB junior share in BPS when receiver=BOB: 9999
  31. CHARLIE junior share in BPS when receiver=BOB: 0
  32. BOB redeemable value after revenue: 51817158902
  33. CHARLIE redeemable value after revenue: 14

[PASS] test_PoC_AssignToTreasuryDilutesAllExistingLpsToZero() (gas: 14849472)
Logs:
  01. junior principal after LP deposits: 20000999316
  02. junior principal after loss: 0
  03. junior supply after loss: 20014328872699
  04. BOB shares after loss: 10006664431847
  05. CHARLIE shares after loss: 10006664440852
  06. senior principal after loss: 79818160005
  07. senior HWM after loss: 80001001089
  08. senior HWM gap to restore: 182841084
  10. estimateDepositShares(1000 USDG) at zero NAV: 0
  11. quoteBootstrapDeposit(1000 USDG) at zero NAV: 20014328873699000000000
  12. senior principal after recap: 80001001089
  13. unassignedAssets after recap: 1817158916
  14. junior principal after assign: 1817158916
  15. junior supply after assign: 36369216180612704622983
  16. BOB shares after assign: 10006664431847
  17. CHARLIE shares after assign: 10006664440852
  18. treasury shares after assign: 36369216160598375750284
  19. BOB junior share in BPS: 0
  20. CHARLIE junior share in BPS: 0
  21. treasury junior share in BPS: 9999
  22. junior principal after revenue: 51817158916
  23. BOB redeemable value after revenue: 14
  24. CHARLIE redeemable value after revenue: 14
  25. treasury redeemable value after revenue: 51817158888
  26. paid to BOB in redeem epoch: 8
  27. paid to treasury in redeem epoch: 31815082636

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 20.42ms (32.69ms CPU time)
```

