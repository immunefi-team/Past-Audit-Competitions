# 57969 sc insight lack of incentive to liquidate small positions can cause the system to accumulate bad debt

**Submitted on Oct 29th 2025 at 17:32:00 UTC by @Oxdeadmanwalking for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57969
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The Alchemist, allows positions to be liquidated below a minimum collateralization threshold. This liquidation is always partial, aiming to restore the position's collateralization to a targer level. Any user is allowed to open as many positions as they wish, without size constraints, with each position being identified by a unique NFT id. The lack of minimum size however, can make liquidations unprofitable for very small positions due to gas costs, especially on ETH mainnet. Since an account's positions are isolated, a malicious actor can open a large amount of small positions and in the event that they become liquidatable, the lack of incentive to liquidate can cause bad debt to the protocol.

## Vulnerability Details

Upon borrowing in Alchemist, no minimum amount is enforced, only a non 0 amount and the id of the position is user supplied, with 0 indicating creation of a new isolated position of arbitrary size

```
    function mint(uint256 tokenId, uint256 amount, address recipient) external {
        _checkArgument(recipient != address(0));
        _checkForValidAccountId(tokenId);
@>  _checkArgument(amount > 0);
        _checkState(!loansPaused);
        _checkAccountOwnership(IAlchemistV3Position(alchemistPositionNFT).ownerOf(tokenId), msg.sender);

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before more is taken
        _sync(tokenId);

        // Mint tokens to recipient
        _mint(tokenId, amount, recipient);
    }
```

The liquidation mechanism also does not check for a minimum size when liquidating, meaning all positions below the minimum collateralization, no matter how small can get liquidated, which is intended.

```
        // Recalculate ratio after any repayment to determine if further liquidation is needed
        collateralInUnderlying = totalValue(accountId);
        collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;
        if (collateralizationRatio <= collateralizationLowerBound) {
            // Do actual liquidation
            return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
        } else {
            // Case: debt still exists but repayment brought the account back to normal levels
            // Since only a repayment happened, send repayment fee to caller
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
```

A malicious actor however can start opening a large amount of small positions denominated in MYT and wait for the positions to be liquidatable. Due to the nature of the liquidation system, the fee paid to the liquidator will not be sufficient to cover the gas costs, making liquidations unprofitable. As MYT drops in price, those liquidatable positions will accumulate until bad debt accumulates in the system, potentially causing the protocol to be insolvent.

Lastly, the protocol has no mention of a protocol owned liquidation bot that will handle those small liquidations.

## Impact Details

Lack of incentive to liquidate small positions can cause the protocol to incur bad debt and potentially go insolvent. A minimum debt amount is advised per-nft position to avoid this.

## References

* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L791>
* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L418>

## Proof of Concept

## Proof of Concept

1. Add this test to the end of `Alchemist.t.sol`

```
    function test_POC_small_positions_uneconomical_to_liquidate() external {
        // Setup healthy global position first (ensures good global collateralization)
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), 10000 ether);
        alchemist.deposit(10000 ether, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // Setup: Create a small position 1e18 worth of MYT
        uint256 smallDeposit = 1 ether;

        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), smallDeposit);
        alchemist.deposit(smallDeposit, externalUser, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));

        // Borrow max (90% LTV = 0.9 ETH)
        uint256 maxBorrow = alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenId, maxBorrow, externalUser);
        vm.stopPrank();

        // Make position undercollateralized by dropping collateral value 6%
        // Less severe drop ensures position has surplus for fee payment
        uint256 initialSupply = IERC20(mockStrategyYieldToken).totalSupply();
        uint256 newSupply = initialSupply * 106 / 100;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(newSupply);

        // Measure gas cost of liquidation
        address liquidator = address(0x999);
        vm.startPrank(liquidator);
        vm.startSnapshotGas("liquidate_small_position");
        (uint256 assets, uint256 feeInYield,) = alchemist.liquidate(tokenId);
        uint256 gasUsed = vm.stopSnapshotGas("liquidate_small_position");
        vm.stopPrank();


        console.log("----------------Liquidation completed----------------");
        console.log("gasUsed: ", gasUsed);
        console.log("feeInYield: ", feeInYield);
        console.log("assets liquidated: ", assets);
    }
```

2. Run the test:

```
forge test --match-test test_POC_small_positions_uneconomical_to_liquidate -vv
```

3. The output should look like the following

```
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] test_POC_small_positions_uneconomical_to_liquidate() (gas: 1293868)
Logs:
  ----------------Liquidation completed----------------
  gasUsed:  319221
  feeInYield:  1379999999999999
  assets:  553799999999999994
```

We can see that for \~0,5 MYT seized, the fee was only \~0,13MYT . Assuming a price of MYT of 1.1USD due to yield accrued in the share, gas costs of 10 gwei andf ETH at 4000 USD, the total cost of liquidation given the gas measurement in the test is:

gasUsed \* ETH per gas (gwei) \* ETH Price = 319221 \* 0.000000001 \* 4000 = \~1,27 USD.

So in the end:

* totalCost =\~1,27 USD.
* totalIncentive \~= 0,13 \* 1,1 = 0,143 USD

Making small positions unprofitable to liquidate.
