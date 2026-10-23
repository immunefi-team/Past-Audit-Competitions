# 56846 sc medium liquidation will return because of insufficient funds

**Submitted on Oct 21st 2025 at 07:22:43 UTC by @pashap9990 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56846
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

### Finding Description and Impact

CPD holders' positions are prone to liquidation upon their CR falling below `collateralizationLowerBound`. Also, the earmark will be paid first, and then the liquidator will get a reward fee once the debt is gone. Nevertheless, the liquidation transaction will fail due to insufficient funds, resulting in a denial of service.

### Textual PoC

Presume `collateralizationLowerBound` is `1.05`

* User A deposits 110 MYT and mints 100 alUSD. \[collateral's worth = 110, debt = 100, earmark = 0] A redemption will be initiated by `Transmuter` with a redeem amount of 100 alUSD, reflecting user A's status after 5,256,000 blocks. \[collateral's worth = 110, debt = 100, earmark = 100]
* The price of MYT has decreased by 20%, indicating the status of user A. \[collateral's worth = 91, debt = 100, earmark = 100]
* A liquidator identifies the opportunity and initiates a liquidation transaction. `_forceReapy` will be invoked as user A possesses an earmark, resulting in the transfer of 110 MYT to `Transmuter`. The liquidate function subsequently attempts to transfer 1.1 MYT to the liquidator as a reward. However, the transaction will fail due to insufficient funds.

### Code Snippet

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L826C24-L826C66>

## Proof of Concept

## Proof of Concept

Kindly incorporate the following PoC in `Alchemist.t.sol`

```solidity
    function testLiquidationWillRevertDueToLackOfFunds() public {
        vm.prank(alOwner);
        alchemist.setProtocolFee(protocolFee);
        
        
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(110e18, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, 99e18 , address(0xbeef));
        vm.stopPrank();


        vm.startPrank(address(0xdad));
        deal(address(alToken), address(0xdad), 99e18);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 99e18);
        transmuterLogic.createRedemption(99e18);
        vm.stopPrank();

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));
        vm.roll(block.number + 5_256_000);
        (uint collateral, uint debt, uint earmark) = alchemist.getCDP(tokenIdFor0xBeef);

        // // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 20 bps or 20%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 2000 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        uint totalValue = alchemist.totalValue(tokenIdFor0xBeef);
        vm.expectRevert();
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);


    }
```
