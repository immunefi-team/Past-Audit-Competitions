# 58616 sc medium liquidation can revert due to 0 amount fee withdraw&#x20;

**Submitted on Nov 3rd 2025 at 15:53:38 UTC by @JoeMama for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58616
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

During liquidation, it first attempts to use the user’s earmark to repay as much debt as possible.

There is a problem when the earmark is close to the debt, it does not wipe out the debt completely and leave a small debt.

Now, the outsourcedFee (calculated from the remaining debt) can become extremely small.

If the underlyingConversionFactor is larger than the outsourcedFee, the normalized value in `normalizeDebtTokensToUnderlying(outsourcedFee)` may round down to zero (e.g., 5e11 / 1e12 = 0).

This results in the fee vault’s withdraw to revert, since it fails the `_checkNonZeroAmount` validation that prevents 0 withdrawals.

```
    function withdraw(address recipient, uint256 amount) external override onlyAuthorized {
        _checkNonZeroAddress(recipient);
        _checkNonZeroAmount(amount);
```

## Vulnerability Details

The root cause of the issue is that the withdraw on the fee vault reverts if the amount is 0, this is caused by liquidating a low debt, because the outsource fee which is based on a percentage of the **new** debt, which will be divided by the `underlyingConversionFactor`,

```
        if (outsourcedFee > 0) { 
            uint256 vaultBalance = IFeeVault(alchemistFeeVault).totalDeposits();
            if (vaultBalance > 0) {
              
                uint256 feeBonus = normalizeDebtTokensToUnderlying(outsourcedFee); 
             
                feeInUnderlying = vaultBalance > feeBonus ? feeBonus : vaultBalance;
                IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying);  // this returned 0. 
            }
        }
```

A low debt could be created when the earmark is removed during forced repayment, so if a user has a large amount of debt before, this could become dust debt. It can also happen when minting a lot of small debts

## Impact Details

This issue can temporarily cause liquidations to fail when a user’s remaining debt is very small.

It can occur naturally, but a user could also intentionally create tiny debts to avoid liquidation.

## Remediation

Consider changing the check from:

```
 if (outsourcedFee > 0) { 
```

to:

```
 if (normalizeDebtTokensToUnderlying(outsourcedFee) > 0) { 
```

## Link to Proof of Concept

<https://gist.github.com/hexens-joe/dce396b3a6473beac2b8c0e6e45785b3>

## Proof of Concept

## Proof of Concept

1. Alice her current output for getCDP are as follow:

```
"collateral", 13000000000000000 
"debt", 11700000000000000
"earmark", 11682913738657214
```

2. Someone calls liquidate on Alice,

the earmark removes 11682913738657214 \[1.168e16] debt. leaving alice with dust debt: 17086261342786 \[1.708e13])

this will return a outsource fee of:

`17086261342786 *feeBps/BPS = 512587840283 [5.125e11])`

**For a 6 decimal vault with a underlyingConversionFactor of 1e12** the fee bonus will be normalizeDebtTokensToUnderlying(5.125e11) = 0 causing the withdraw to **revert** with `ZeroAmount()` error.

PoC:

Please run the gist with: `forge test --mt testLiquidateReverts -vv`

```
function testLiquidateReverts() external {

        address user1 = makeAddr("user1");
        address user2 = makeAddr("user2");

        deal(address(vault), address(user1), 10e18);

        vm.startPrank(address(user1));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(10e18, address(user1), 0); 
        vm.stopPrank();


        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(user1), address(alchemistNFT));

        vm.prank(user1);
        alchemist.mint(tokenId, 9e18, user1);

        vm.startPrank(user1);

        IERC20(alToken).approve(address(transmuterLogic), 3000e18);
        IERC20(address(vault)).approve(address(alchemist), 100_000e18);
        transmuterLogic.createRedemption(9e18 - 100);

        vm.roll(vm.getBlockNumber() + transmuterLogic.timeToTransmute()); // full dulration of the redemption.

        IERC20(address(vault)).approve(address(alchemist), 100_000e18);
       
        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 2000 bps or 0% while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = ((initialVaultSupply * 2000) / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        vm.startPrank(user1);
        vm.expectRevert("ZeroAmount()");
        alchemist.liquidate(tokenId);

    }

```
