# 58754 sc high missing mytsharesdeposited decrements in alchemistv3 forcerepay doliquidation&#x20;

**Submitted on Nov 4th 2025 at 11:53:02 UTC by @oct0pwn for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58754
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The \_forceRepay and \_doLiquidation functions in AlchemistV3 fail to decrement \_mytSharesDeposited after MYT tokens are repaid or liquidated. This results in the system accounting showing higher than actual deposited MYT shares, leading to an inflated perception of collateral coverage. Consequently, redeemers or liquidators may receive more principal than they are entitled to, effectively causing principal overpayment and system-level accounting imbalance.

## Vulnerability Details

The AlchemistV3 contract tracks MYT deposits using an internal accounting variable \_mytSharesDeposited, which represents the total MYT shares deposited into the system. When MYT tokens are withdrawn, repaid, or liquidated, the corresponding amount should be **decremented** from \_mytSharesDeposited to maintain consistent accounting.

However, in both \_forceRepay() and \_doLiquidation() functions, the logic handles MYT transfers and balance updates but does not update \_mytSharesDeposited, as seen below:

```solidity
if (creditToYield > 0) {
    // Transfer the repaid tokens from the account to the transmuter.
    TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
}
```

This omission causes \_mytSharesDeposited to remain unchanged even though MYT tokens are effectively moved out of the strategy’s control during a repayment or liquidation. As a result, the system incorrectly believes more MYT is still backing outstanding liabilities than actually exists.

In scenarios involving redemptions or liquidations, this inflated value of \_mytSharesDeposited can cause miscalculations in yield accounting and redemption share ratios — allowing redeemers to receive more principal value than intended.

Proper accounting requires decrementing \_mytSharesDeposited whenever MYT is transferred out from the protocol’s collateral reserves:

```solidity
_mytSharesDeposited -= creditToYield; // Adjust for repaid MYT
```

## Impact Details

Failure to decrement \_mytSharesDeposited leads to **collateral accounting inflation**, resulting in:

* **Principal overpayment risk:** Redeemers or liquidators may receive a higher principal value due to overstated share accounting.
* **Inaccurate system metrics:** The protocol will report a higher-than-actual deposited MYT balance, distorting health factor and solvency calculations.
* **Long-term drift in system invariants:** Repeated repayments and liquidations will compound the error, creating persistent imbalances between real collateral and tracked deposits.
* **Potential protocol losses:** Over-redemption or excessive liquidation payouts can result in a net loss of protocol-held assets or depletion of reserves.

## Mitigation

To ensure correct collateral and share accounting, the contract should **decrement \_mytSharesDeposited** whenever MYT tokens are transferred out of the system through repayment or liquidation. This ensures that the internal state remains consistent with the actual MYT balance held by the protocol.

Update both \_forceRepay() and \_doLiquidation() as follows:

```solidity
// _forceRepay()
if (creditToYield > 0) {
    // Transfer the repaid tokens from the account to the transmuter.
    TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);

    // ✅ Adjust internal accounting to reflect decreased MYT holdings
    _mytSharesDeposited -= creditToYield;
}

// _doLiquidation()
if (liquidatedAmount > 0) {
    TokenUtils.safeTransfer(myt, address(transmuter), liquidatedAmount);

    // ✅ Decrement MYT shares deposited
    _mytSharesDeposited -= liquidatedAmount;
}
```

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

Place the following test in `AlchemistV3.t.sol`:

```solidity
 function testAudit_MytSharesDepositedNotDecrementedOnLiquidation() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // Setup: Create healthy account to maintain system collateralization
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(
            address(vault),
            address(alchemist),
            depositAmount * 2
        );
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // Setup: Create undercollateralized position
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(
            address(vault),
            address(alchemist),
            depositAmount + 100e18
        );
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(
            address(0xbeef),
            address(alchemistNFT)
        );
        alchemist.mint(
            tokenId,
            (alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR) /
                minimumCollateralization,
            address(0xbeef)
        );
        vm.stopPrank();

        // Drop yield token price to make position undercollateralized
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken))
            .totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(
            initialVaultSupply
        );
        uint256 modifiedVaultSupply = ((initialVaultSupply * 590) / 10_000) +
            initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(
            modifiedVaultSupply
        );

        // Record state before liquidation
        uint256 beforeBalance = IERC20(address(vault)).balanceOf(
            address(alchemist)
        );
        uint256 beforeTVL = alchemist.getTotalUnderlyingValue();

        // Perform liquidation (transfers MYT out but doesn't decrement _mytSharesDeposited)
        vm.startPrank(externalUser);
        alchemist.liquidate(tokenId);
        vm.stopPrank();

        // Record state after liquidation
        uint256 afterBalance = IERC20(address(vault)).balanceOf(
            address(alchemist)
        );
        uint256 afterTVL = alchemist.getTotalUnderlyingValue();

        // Calculate how much MYT was actually transferred out
        uint256 mytTransferredOut = beforeBalance - afterBalance;
        uint256 expectedTVLDecrease = alchemist.convertYieldTokensToUnderlying(
            mytTransferredOut
        );
        uint256 actualTVLDecrease = beforeTVL - afterTVL;

        console.log("MYT balance before liquidation:", beforeBalance);
        console.log("MYT balance after liquidation:", afterBalance);
        console.log("MYT transferred out:", mytTransferredOut);
        console.log(
            "Expected TVL decrease (based on MYT transferred):",
            expectedTVLDecrease
        );
        console.log("Actual TVL decrease:", actualTVLDecrease);
        console.log("Mismatch (bug):", expectedTVLDecrease - actualTVLDecrease);

        // BUG PROOF: TVL doesn't decrease by the amount of MYT transferred out
        // because _mytSharesDeposited wasn't decremented
        assertLt(
            actualTVLDecrease,
            expectedTVLDecrease,
            "BUG: TVL decrease is less than expected - _mytSharesDeposited not decremented during liquidation"
        );
    }
    
    
    
// EXPECTED OUTPUT:
//  MYT balance before liquidation: 400000000000000000000000
//  MYT balance after liquidation: 290985999999999998125641
//  MYT transferred out: 109014000000000001874359
//  Expected TVL decrease (based on MYT transferred): 102940509915014165977264
//  Actual TVL decrease: 0
//  Mismatch (bug): 102940509915014165977264

// Suite result: ok. 1 passed; 0 failed; 0 skipped;
```
