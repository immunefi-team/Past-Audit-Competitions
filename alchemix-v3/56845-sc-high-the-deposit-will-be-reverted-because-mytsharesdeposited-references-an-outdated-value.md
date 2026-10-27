# 56845 sc high the deposit will be reverted because mytsharesdeposited references an outdated value

**Submitted on Oct 21st 2025 at 07:19:02 UTC by @pashap9990 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56845
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Finding Description and Impact

`_mytSharesDeposited` shows how much collateral has been put into `AlchemistV3`. In addition, its value will go down once CDP holders take back their collateral. Nevertheless, it would not update after a CDP that will be liquidated, leading to denial of service.

Furthermore, users are prohibited from depositing into `AlchemistV3` beyond the established capacity set by the owner. This implies that if `_mytSharesDeposited` exceeds the actual amount, the user cannot deposit, as `_mytSharesDeposited + amount` would surpass `depositCap`.

### Textual PoC

Presume `depositCap` is 200,000

* User A deposits 200,000 alUSD into AlchemistV3.(\_mytSharesDeposited = 200,000)
* User A will be susceptible to liquidation, resulting in the transfer of their collateral to `Transmitur`, while `_mytSharesDeposited` remains at 200,000.
* Other users are unable to deposit, as AlchemistV3's capacity is at its limit.

### Code Snippet

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L549>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L369>

## Proof of Concept

## Proof of Concept

Please incorporate the following PoC in `AlchemistV3.t.sol`.

```solidity
    function testMytSharesDepositedPointsToStaleValueWhichCausesProtocolInsolvency() public {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.prank(alOwner);
        alchemist.setDepositCap(depositAmount);



        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
        
        
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);


        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount / 2);
        vm.expectRevert();
        alchemist.deposit(depositAmount / 2, yetAnotherExternalUser, 0);
        vm.stopPrank();
        
    }
```
