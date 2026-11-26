# \#98314 [SC-Insight] Direct donations to `TrancheVault` remain unaccounted and cannot be recovered, unlike `HousePool`

**Submitted on Nov 6th 2026 at 23:14:01 UTC by @blackgrease for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98314
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

## Brief/Intro

`TrancheVault` does not account for USDG transferred directly to its address, nor does it provide a function to recover those funds. The tokens remain in the vault without increasing LP assets or any user's deposit entitlement. Comparatively, unsolicited or accidental transfers to HousePool can be recovered by the Protocol.

## Vulnerability details

`TrancheVault` has no way to recover donations accidentally sent into it. Donations do not affect accounting because `TrancheVault.totalAssets()` obtains the tranche principal from `HousePool`, rather than using the vault's token balance. Only deposits made through `requestDeposit()` increase `pendingDepositEscrowAssets`, while redemption funding increases `withdrawalEscrowAssets`. Because a direct donation updates neither amount, it has no effect on accounting and stays stranded in the vault.

- `TrancheVault.totalAssets()`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L510-L513
- `TrancheVault._requestDeposit()`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L577-L603
- `TrancheVault.fundRedeemEpoch()`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1478-L1499

### Inconsistency with HousePool

`HousePool` explicitly handles unsolicited USDG transfers by:
1. detecting them through `excessAssets()`, while `totalAssets()` ignores them
2. and importantly they can be recovered by the owner through `sweepExcess()`.

`TrancheVault` has no equivalent mechanism, even though it also holds USDG during deposit and redemption workflows. 

The protocol documents this limitation for the Clearinghouse (SECURITY.md "Clearinghouse limitations"), but not for `TrancheVault`, whose sibling `HousePool` does provide a recovery path.

### Example Scenario

1. A user mistakenly transfers 25,000 USDG directly to `juniorVault` instead of calling `requestDeposit()`.
2. The vault receives the USDG, but its `totalAssets()` and recorded escrow balances do not change.
3. The user receives no shares or deposit claim. Unlike a direct donation to `HousePool`, the funds have no administrative recovery route.

## Impact

**Code Optimizations and Enhancements:** Unsolicited or accidental USDG donations can remain indefinitely in `TrancheVault` without being credited or recoverable. Because TrancheVault is non-upgradeable, the funds are permanently stranded with no way to return them to the sender.

## Recommendation

Consider introducing an owner-controlled function to recover USDG held above the vault's recorded obligations (`pendingDepositEscrowAssets + withdrawalEscrowAssets`), similar to `HousePool.sweepExcess()`. The function should never allow withdrawal of funds reserved for pending deposits or redemptions.

Alternatively, publicly document that USDG sent directly to `TrancheVault` is unrecoverable, so users and auditors are aware of it.


## Proof of Concept

The provided Foundry test uses the existing `BasePerpTest` fixture and demonstrates:

- A direct USDG transfer to `TrancheVault` increases its raw balance but not its accounted assets or escrow balances.
- The same transfer to `HousePool` is detected as excess and can be swept by the owner without changing pool accounting.
- Alternatively, `HousePool`'s owner can explicitly account for the donated tokens.

Place `PoC_TrancheVaultDonationComparedToHousePool.t.sol` in `packages/perps/test/perps/spec/lp/` and run from the project root:

```bash
forge test --match-contract PoC_TrancheVaultDonationComparedToHousePool --root packages/perps --threads 1 -vvvv
```

### The Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Place in: packages/perps/test/perps/spec/lp/PoC_TrancheVaultDonationComparedToHousePool.t.sol
// Uses Plether's existing production-stack BasePerpTest fixture; no protocol changes.
import {BasePerpTest} from "test/perps/BasePerpTest.sol";
import {console2} from "forge-std/console2.sol";

contract PoC_TrancheVaultDonationComparedToHousePool is BasePerpTest {
    address internal constant DONOR = address(0xD0A0);
    address internal constant RECIPIENT = address(0xBEEF);
    uint256 internal constant DONATION = 25_000e6; // 25,000 USDG

    function setUp() public override {
        super.setUp();
        settlementToken.mint(DONOR, DONATION);
    }

    function testPoC_TrancheVaultDonationDoesNotEnterAccounting() public {
        uint256 rawBefore = settlementToken.balanceOf(address(juniorVault));
        uint256 vaultAssetsBefore = juniorVault.totalAssets();
        uint256 poolAssetsBefore = pool.totalAssets();
        uint256 depositEscrowBefore = juniorVault.pendingDepositEscrowAssets();
        uint256 withdrawalEscrowBefore = juniorVault.withdrawalEscrowAssets();

        // No requestDeposit: the donor sends USDG directly to the tranche vault.
        vm.prank(DONOR);
        assertTrue(settlementToken.transfer(address(juniorVault), DONATION));

        assertEq(settlementToken.balanceOf(address(juniorVault)), rawBefore + DONATION);
        assertEq(juniorVault.totalAssets(), vaultAssetsBefore, "donation must not increase LP assets");
        assertEq(pool.totalAssets(), poolAssetsBefore, "donation does not reach the pool");
        assertEq(juniorVault.pendingDepositEscrowAssets(), depositEscrowBefore, "not a recorded deposit");
        assertEq(juniorVault.withdrawalEscrowAssets(), withdrawalEscrowBefore, "not a funded redemption");

        // The vault has no counterpart to HousePool.accountExcess()/sweepExcess().
        // The extra balance is not assigned to any depositor, LP, or escrow obligation.
        console2.log("TrancheVault unaccounted USDG:", DONATION);
    }

    function testControl_HousePoolDonationCanBeSwept() public {
        uint256 rawBefore = pool.rawAssets();
        uint256 excessBefore = pool.excessAssets();
        uint256 assetsBefore = pool.totalAssets();

        vm.prank(DONOR);
        assertTrue(settlementToken.transfer(address(pool), DONATION));

        assertEq(pool.rawAssets(), rawBefore + DONATION);
        assertEq(pool.excessAssets(), excessBefore + DONATION, "house pool detects excess");
        assertEq(pool.totalAssets(), assetsBefore, "unaccounted donation does not change LP backing");

        vm.prank(pool.owner());
        pool.sweepExcess(RECIPIENT, DONATION);

        assertEq(settlementToken.balanceOf(RECIPIENT), DONATION, "owner can recover the excess");
        assertEq(pool.excessAssets(), excessBefore, "excess is removed");
        assertEq(pool.totalAssets(), assetsBefore, "legitimate pool accounting is preserved");
    }

    function testControl_HousePoolDonationCanBeExplicitlyAccounted() public {
        uint256 excessBefore = pool.excessAssets();
        uint256 assetsBefore = pool.totalAssets();

        vm.prank(DONOR);
        assertTrue(settlementToken.transfer(address(pool), DONATION));

        assertEq(pool.excessAssets(), excessBefore + DONATION);
        assertEq(pool.totalAssets(), assetsBefore);

        vm.prank(pool.owner());
        pool.accountExcess();

        // Existing setup has no unaccounted pool excess; the exact donation is admitted.
        assertEq(pool.excessAssets(), 0, "accounted excess is cleared");
        assertEq(pool.totalAssets(), assetsBefore + DONATION, "owner can recognize excess as pool backing");
    }
}
```



