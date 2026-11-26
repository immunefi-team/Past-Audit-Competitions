# \#100064 [SC-Critical] assignUnassignedAssets into a wiped tranche mints ~the whole share supply to the receiver, permanently confiscating surviving wiped-tranche LPs' recovery entitlement

**Submitted on Nov 16th 2026 at 07:08:21 UTC by @niroh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100064
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Permanent freezing of unclaimed yield

## Description

## Brief/Intro
The `HousePool` contract has an owner-only cleanup function, `assignUnassignedAssets`, that hands "ownerless" quarantined cash to an LP tranche by minting matching shares. When the target tranche has been wiped out by trading losses (its principal is zero but its share supply still exists from the seed floor and surviving LPs), the share pricing treats the tranche as if it were brand new. A receiver assigned even **1 USDC** of ownerless cash is minted shares equal to roughly a million times the entire surviving share supply - i.e. essentially 100% of the tranche. Every surviving LP of that wiped tranche permanently loses their pro-rata claim on the tranche's future recovery revenue (for Senior, the HWM restoration stream), which is instead redirected to the receiver. The damage occurs regardless of who the receiver is, and it bypasses the deposit gate that normally forbids anyone from entering a wiped tranche.

## Vulnerability Details
The flaw is a **share-pricing / allocation bypass** in the bootstrap assignment path.

**Affected code:**

- [`packages/perps/src/HousePool.sol:660-703`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L660-L703) - `assignUnassignedAssets`. Lines 685-696 quote the mint and apply the senior special-case:

```solidity
685  uint256 shares = ITrancheVaultBootstrap(targetVault).quoteBootstrapDeposit(amount);
686  if (shares == 0) {
687      revert HousePool__BootstrapSharesZero();
688  }
...
// senior branch (lines ~690-696):
if (seniorPrincipal == 0) {
    seniorPrincipal = amount;
    seniorHighWaterMark = amount;   // hard-set, overwrites any surviving HWM
} else {
    ...
}
```

- [`packages/perps/src/TrancheVault.sol:1392-1396`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1392-L1396) - `quoteBootstrapDeposit`, and `1848-1860` - `_depositPricingAssets()` / `_convertToSharesUsingAssets()`. The pricing base is the tranche's **principal**; when it is zero the conversion degenerates to:

```
shares = assets * (accruedSupply + 1000) / (0 + 1)  ≈  assets × entireSupply
```

- [`packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:165-178`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L165-L178) - `absorbLoss` zeroes a tranche's principal on loss but deliberately leaves the share supply and senior high-water mark intact; the survivors' remaining value is exactly the future recovery claim.

- [`packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:104-135`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L104-L135) - `applyClaimantRecapitalizationIntent`. When senior has owners and `seniorPrincipal != 0`, only the HWM gap is restored; with gap == 0 the **entire** recapitalization amount is routed to `unassignedAssets` (lines ~118-135). This is how the trigger state is produced by a documented trusted action (owner recapitalization).

**Faulty assumption:** the bootstrap path assumes an assigned-to tranche is either healthy or empty-and-ownerless. It performs none of the HWM-gap / pro-rata routing that the revenue and recapitalization flows deliberately implement for impaired tranches, and it does not guard against minting into a wiped-but-populated tranche. Note that ordinary deposits into a wiped tranche are correctly blocked (`TrancheVault._isTerminallyWiped` / `_projectedTerminalWipe` cause `TrancheVault__TerminallyWiped` / `TrancheVault__DepositsUnavailable` reverts and `maxRequestDeposit == 0`) - the bootstrap path bypasses exactly that protection.



## Impact Details
Maps to permanent freezing of unclaimed yield (high)

What is confiscated is not cash already accrued, but the wiped-tranche LPs' **permanent pro-rata entitlement to future recovery revenue** routed to that tranche:

- Junior case: after assigning `U` USDC of ownerless residue, the receiver owns `U/(1+U)` of the tranche (1 USDC → 99.9999%). All future junior revenue `R` then accrues pro-rata; incumbents retain only `1/(1+U)` of their prior recovery claim. In the PoC, 50,000 USDC of recovery revenue that would have been worth ≈49,505 USDC to the surviving LP is instead almost entirely captured by a receiver who contributed 1 USDC of ownerless cash.
- Senior case (not PoC-exercised): the `seniorPrincipal == 0` branch hard-sets `seniorHighWaterMark = amount`, destroying the surviving HWM restoration target `H`, and the receiver captures the future senior-restoration stream.

The trigger requires an owner recapitalization plus an owner `assignUnassignedAssets` call - both documented trusted actions - but the harm does **not** depend on the receiver being malicious; a benign cleanup address causes the same confiscation. Magnitude scales with the amount of future recovery revenue routed to the wiped tranche; in a realistic post-loss recovery scenario this is the entire tranche's recovery value.

## Proof of Concept

## Proof of Concept
Repo: `immunefi-team-audit-comp-plether`, frozen commit `a82d4fef…` (per audit scope).
Written against the repo Foundry suite (`BasePerpTest`), no fork required; mocks limited to Pyth/USDG per the standard test harness.

Setup:
```bash
git checkout a82d4fef
forge install
forge test --match-contract WipedTrancheAssignDilutionPoC -vvv
```

`test/perps/spec/lp/WipedTrancheAssignDilutionPoC.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

contract WipedTrancheAssignDilutionPoC is BasePerpTest {

    address alice = address(0xA11CE); // surviving junior LP
    address carol = address(0xCA201); // senior LP
    address receiver = address(0x5EC0); // benign receiver of the owner cleanup

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _reconcile() internal {
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function _addRevenue(uint256 amount) internal {
        settlementToken.mint(address(pool), amount);
        pool.accountExcess();
        _reconcile();
    }

    function test_AssignIntoWipedJuniorConfiscatesIncumbentRecovery() public {
        _fundJunior(alice, 100_000e6);
        _fundSenior(carol, 100_000e6);
        _reconcile();

        // 1. Junior is wiped by a loss equal to its principal; senior stays whole (gap == 0).
        uint256 jp = pool.juniorPrincipal();
        // Model a realized trader profit paid out of the pool by the engine (accountedAssets and cash both drop).
        vm.prank(address(engine));
        pool.payOut(address(0xdead), jp);
        _reconcile();
        assertEq(pool.juniorPrincipal(), 0, "junior wiped");
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "senior healthy, no HWM gap");
        uint256 juniorSupply = juniorVault.totalSupply();
        assertGt(juniorSupply, 0, "junior supply survives wipe");
        uint256 aliceShares = juniorVault.balanceOf(alice);
        console2.log("junior supply after wipe", juniorSupply);
        console2.log("alice junior shares", aliceShares);
        console2.log("ordinary junior maxRequestDeposit while wiped", juniorVault.maxRequestDeposit(carol));

        // 2. Owner/engine recapitalization: gap == 0 so the whole amount becomes unassignedAssets.
        uint256 U = 1e6; // 1 USDC
        settlementToken.mint(address(pool), U);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            U, IHousePool.ClaimantInflowKind.Recapitalization, IHousePool.ClaimantInflowCashMode.CashArrived
        );
        _reconcile();
        assertEq(pool.unassignedAssets(), U, "recap residue routed to unassigned");
        assertEq(pool.juniorPrincipal(), 0);

        uint256 R = 50_000e6; // future trading revenue recovering junior

        // Counterfactual: no assignment, revenue arrives.
        uint256 snap = vm.snapshotState();
        _addRevenue(R);
        uint256 aliceValueNoAssign = juniorVault.convertToAssets(aliceShares);
        console2.log("junior principal after revenue (no assign)", pool.juniorPrincipal());
        console2.log("alice value w/o assignment", aliceValueNoAssign);
        vm.revertToState(snap);

        // 3. Owner performs the documented cleanup to a benign receiver.
        uint256 quoted = juniorVault.quoteBootstrapDeposit(U);
        pool.assignUnassignedAssets(false, receiver);
        uint256 minted = juniorVault.balanceOf(receiver);
        assertEq(minted, quoted);
        console2.log("bootstrap shares minted for 1 USDC", minted);
        console2.log("receiver fraction (bps)", minted * 10_000 / juniorVault.totalSupply());
        // shares = U * (supply + 1000) / 1  -> ~1e6x the surviving supply
        assertGe(minted, U * juniorSupply, "receiver minted ~U x whole supply");

        // 4. Same future revenue now accrues pro-rata to the receiver.
        _addRevenue(R);
        uint256 aliceValueAssign = juniorVault.convertToAssets(aliceShares);
        uint256 receiverValue = juniorVault.convertToAssets(minted);
        console2.log("junior principal after revenue (assign)", pool.juniorPrincipal());
        console2.log("alice value with assignment", aliceValueAssign);
        console2.log("receiver value (paid 0, ownerless 1 USDC)", receiverValue);

        assertGt(aliceValueNoAssign, 40_000e6, "w/o assignment the incumbents recover the revenue");
        assertLt(aliceValueAssign, 1e6, "with assignment alice recovers < 1 USDC of 50k revenue");
        assertGt(receiverValue, 49_000e6, "receiver captures the recovery");
    }

}
```

**Expected output:** the test passes with logs showing:

- `junior supply after wipe` ≈ `1.01e14` (surviving supply), while `ordinary junior maxRequestDeposit while wiped == 0` - confirming the wiped tranche is closed to ordinary entry.
- `bootstrap shares minted for 1 USDC` ≈ `1.01e20` (`U × (supply + 1000)`), receiver fraction ≈ `9999 bps` (≈99.99%).
- `alice value w/o assignment` ≈ `49,505e6` vs `alice value with assignment` `< 1e6`; `receiver value` ≈ `50,000.95e6` for a receiver who contributed nothing but received the 1 USDC ownerless assignment.

**Why this demonstrates the bug:** `quoteBootstrapDeposit` prices against the tranche's zero principal, so the mint is `U × entireSupply` instead of a pro-rata amount. The assertion `minted >= U * juniorSupply` proves the whole-supply mint; the counterfactual snapshot comparison proves the surviving LP's recovery claim on the same 50,000 USDC revenue stream collapses from ≈49,505 USDC to a fraction of a cent, purely due to the owner's documented cleanup call.
