# 57553 sc high mytsharesdeposited is not updated in liquidations which breaks bad debt ratio alchemistcr calculations and causes failures in bad debt handling and liquidation handling&#x20;

**Submitted on Oct 27th 2025 at 08:07:52 UTC by @niroh for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57553
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

AlchemistV3 maintains a \_mytSharesDeposited state that tracks the amount of collateral deposited in the system. The state is updated in deposits/withdrawals/redeems and when collateral is paid out as fees. However it is not reduced when collateral is transferred to Transmuter during liquidations.

The value of \_mytSharesDeposited is used in these two places (among other)

1. In transmuter::claimRedemption when calculating the system bad debt ratio so that redemption amounts can be scaled down based on bad debt. (alchemist.getTotalUnderlyingValue() is based on \_mytSharesDeposited):

```solidity
// Ratio of total synthetics issued by the alchemist / underlingying value of collateral stored in the alchemist
// If the system experiences bad debt we use this ratio to scale back the value of yield tokens that are transmuted
uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
// Avoid divide by 0
uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

uint256 scaledTransmuted = amountTransmuted;

if (badDebtRatio > 1e18) {
    scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
}
```

2. In DoLiqquidation/CalculateLiquidation where system CR is checked and if bellow the minimum CR - a full liquidation is run instead of a partial one:

```solidity
(uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
    collateralInUnderlying,
    account.debt,
    minimumCollateralization,
    //AUDIT COMMENT: Alchemist CR calculation
    normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt, 
    globalMinimumCollateralization,
    liquidatorFee
);
```

```solidity
//AUDIT COMMENT - alchemist CR usage in calculateLiquidation:
 if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
    outsourcedFee = (debt * feeBps) / BPS;
    // fully liquidate debt in high ltv global environment
    return (debt, debt, 0, outsourcedFee);
}

```

## Vulnerability Details

Failing to update mytSharesDeposited during liquidations breaks both calculations that rely on it (mentioned above):

1. In transmuter - the bad debt ratio is calculated largely as `totalDebtTokenIssuance / (Alchemist.GetTotalUnderlyingValue() + TransmuterCurrentCollateralAsUnderlying)`. However, AlchemistTotalUnderlyingValue is based on the alchemist \_mytSharesDeposited state. Since this state is not reduced during liquidations, any collateral transfered from alchemist to transmuter during liquidations will be counted twice. This will "overestimate" system health and fail to detect bad debt when it exists. The result is that redemptions will not be scaled down to bad debt, which will cause a "last redeemer loses" situation leading to bank runs and accelerate system involvancy during bad debt periods.
2. Similarly, In DoLiquidation/calculateLiquidation, the CR calc is based on the ratio of \_mytSharesDeposited (in debt token terms) to the Alchemist debt. Here too the CR calculation should not include collateral that was transferred to Transmuter (because only collateral in Alchemist covers alchemist debt). Liquidations will cause this calculation to "overestimate" system health and fail to detect high LTV situations, enabling partial liquidations that increase insolvancy risk during system-high-ltv periods (in addition to under-paying liquidator fees which are supposed to be based on full liquidations in high-ltv scenarios) .

## Impact Details

1. Failure to assign bad debt on redeemers fairly which can lead to bank runs or assign the full bad debt loss to the last redeemers
2. Failure to perform full liquidations in times of systemwide-high-ltv, increasing system risk of insolvancy.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L541C9-L541C29>

## Proof of Concept

## Proof of Concept

How to run:

1. Copy the code below into the IntegrationTest contract is test/IntergrationTest.t.sol
2. Comment out the line `addDepositsToMYT();` in the setup() function (this makes myt price drop simulation easier)
3. add the following import: `import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";`
4. Run with FOUNDRY\_PROFILE=default forge test --fork-url <https://mainnet.gateway.tenderly.co> --match-test testMytSharesDepositedErrorPOC -vvv --evm-version cancun

```solidity
 function depositToAlchemix(uint256 shares, address user) internal {
        vm.startPrank(user);
        IERC20(address(vault)).approve(address(alchemist),shares);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        alchemist.deposit(shares,user, tokenId);
        
        vm.stopPrank();
    }

     function MintOnAlchemix(uint256 toMint, address user) internal {
         vm.startPrank(user);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        assertGt(tokenId,0,"cannot mint to user with no positions");
        alchemist.mint(tokenId, toMint, user);
        vm.stopPrank();
    }

    function RedeemOnTransmuter(address user, uint256 debtAmount) internal {
        vm.startPrank(user);

        IERC20(alUSD).approve(address(transmuterLogic), debtAmount);
        transmuterLogic.createRedemption(debtAmount);

        vm.stopPrank();

    }

    function moveTime(uint256 blocks) internal {
        vm.warp(block.timestamp+blocks*12);
        vm.roll(block.number+blocks);
    }

    function printState(address user, string memory message) internal {
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        //poke to refresh _earmark and sync
        alchemist.poke(tokenId);
        uint256 totalDebt = alchemist.totalDebt();
        uint256 synthIssued = alchemist.totalSyntheticsIssued();
        uint256 transmuterCollBalance = IERC20(address(alchemist.myt())).balanceOf(address(transmuterLogic));
        uint256 transmuterDebtCoverage = alchemist.convertYieldTokensToDebt(transmuterCollBalance);
        uint256 transmuterLocked = transmuterLogic.totalLocked();
        
        (uint256 collateral, uint256 debt, uint256 earmarked) = alchemist.getCDP(tokenId);
        uint256 alchemistCollBalance = IERC20(address(alchemist.myt())).balanceOf(address(alchemist));
        uint256 alchemistCollInDebt = alchemist.convertYieldTokensToDebt(alchemistCollBalance);
        console.log("%s",message);
        console.log("Alchemist Total Debt: %s",totalDebt/1e18);
        console.log("Alchemist cumulative earmarked: %s", alchemist.cumulativeEarmarked() / 1e18);
        console.log("Alchemist getTotalUnderlyingValue (_mytSharesDeposited value in debt tokens)", alchemist.getTotalUnderlyingValue()/1e6);
        console.log("Alchemist Actual Collateral balance value in debt tokens", alchemistCollInDebt / 1e18);
        console.log("Transmuter Debt Coverage (collateral balance value in debt tokens): %s",transmuterDebtCoverage/1e18);
        console.log("Transmuter Locked Synthetic tokens: %s",transmuterLocked/1e18);
        console.log("Total Synthetic token Issuance: %s",synthIssued/1e18);
        
        console.log("CDP info -  collateral: %s, debt %s, earmarked %s\n\n", collateral/1e18,  debt/1e18,  earmarked/1e18);
        
    }

    function ClaimResdemptionTransmuter(address user) internal {
        vm.startPrank(user);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(transmuterLogic));
        transmuterLogic.claimRedemption(tokenId);
        vm.stopPrank();

    }

    function liquidateOnAlchemist(address liquidator, address liquidated) internal {
        vm.startPrank(liquidator);
         uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(liquidated, address(alchemistNFT));
        alchemist.liquidate(tokenId);
        vm.stopPrank();
    }

    function simulateMytPriceDrop(uint256 dropPercent) public {
        uint256 mytVaultUnderlyingBalance  = IERC20(USDC).balanceOf(address(vault));
        vm.prank(address(vault));
        IERC20(USDC).transfer(address(this), mytVaultUnderlyingBalance * dropPercent / 100);
        uint256 sharePrice = VaultV2(vault).convertToAssets(1e18);
        console.log("updated myt share price (underlying tokens per 1e18 myt): %s\n",sharePrice);

    }

    function testMytSharesDepositedErrorPOC() public {
        address bob = makeAddr("bob");
        address redeemer1 = makeAddr("redeemer1");
        address redeemer2 = makeAddr("redeemer2");
        address liquidator = makeAddr("liquidator");

        AlchemistTokenVault alchemistFeeVault = new AlchemistTokenVault(address(vault.asset()), address(alchemist), alOwner);
        vm.startPrank(alOwner);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));
        vm.stopPrank();
         //bob deposit 3000 underlying to vault 
        uint256 sharesBob = _magicDepositToVault(address(vault), bob, 3_000e6);

         //bob deposits 1150 vault shares to alchemix
        uint256 depositedCollateral = 1150e18;
        depositToAlchemix(depositedCollateral, bob);

        //bob mints 1000 alUSD
        uint256 mintAmount = 1000e18;
        MintOnAlchemix(mintAmount,bob);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(bob, address(alchemistNFT));

        //Bob transfers all minted synthetic to 2 redeemers (500 each)
        vm.startPrank(bob);
        IERC20(alUSD).transfer(redeemer1, mintAmount / 2);
        IERC20(alUSD).transfer(redeemer2, mintAmount / 2);
        vm.stopPrank();

        //both redeemers create redemptions for the full amount
        RedeemOnTransmuter(redeemer1, mintAmount / 2);
        RedeemOnTransmuter(redeemer2, mintAmount / 2);
        
        //move to redemption maturity
        moveTime(transmuterLogic.timeToTransmute());
        printState(bob,"state after redeem matured");

        //simulate myt 10% share price drop
        simulateMytPriceDrop(10);

        //liquidate CDP
        liquidateOnAlchemist(liquidator, bob);
        printState(bob, "state after liquidation");
        /*OUTPUT:
        state after liquidation
        Alchemist Total Debt: 0
        Alchemist cumulative earmarked: 0
        Alchemist getTotalUnderlyingValue (_mytSharesDeposited value in debt tokens) 1035
        Alchemist Actual Collateral balance value in debt tokens 15
        Transmuter Debt Coverage (collateral balance value in debt tokens): 999
        Transmuter Locked Synthetic tokens: 1000
        Total Synthetic token Issuance: 1000
        CDP info -  collateral: 16, debt 0, earmarked 0
        
        Note: Almost all colleretal was moved to Transmuter, however getTotalUnderlyingValue still shows 1035.
        */


        //simulate another  20% share price drop (creating bad debt)
        simulateMytPriceDrop(20);
        printState(bob, "state after second (20%) price drop");
        /*
        OUTPUT:
        state after second (20%) price drop
        Alchemist Total Debt: 0
        Alchemist cumulative earmarked: 0
        Alchemist getTotalUnderlyingValue (_mytSharesDeposited value in debt tokens) 828
        Alchemist Actual Collateral balance value in debt tokens 12
        Transmuter Debt Coverage (collateral balance value in debt tokens): 800
        Transmuter Locked Synthetic tokens: 1000
        Total Synthetic token Issuance: 1000
        CDP info -  collateral: 16, debt 0, earmarked 0

        Note: Current Bad Debt Ratio should be: 1000 / (12 + 800) = 1.231 
        but Transmuter will get 1000 / 828 + 800 = 0.614 and therefore will not apply bad debt scaling
        */


        //call claimRedemption
        uint256 balanceBefore = vault.balanceOf(redeemer1);
        ClaimResdemptionTransmuter(redeemer1);
         uint256 balanceDiff = vault.balanceOf(redeemer1)-balanceBefore;
         uint256 balanceDiffValue = alchemist.convertYieldTokensToDebt(balanceDiff);
         console.log("first redemption payment in debt token: %s\n",balanceDiffValue/1e18);
        printState(bob, "state after first redemption claimed");
        

        balanceBefore = vault.balanceOf(redeemer2);
        ClaimResdemptionTransmuter(redeemer2);
         balanceDiff = vault.balanceOf(redeemer2)-balanceBefore;
         balanceDiffValue = alchemist.convertYieldTokensToDebt(balanceDiff);
         console.log("second redemption payment in debt token: %s\n",balanceDiffValue/1e18);
        printState(bob, "state after second redemption claimed");

        //First redemption was not scaled down as it should have been
        //The second redeemer absorbs all bad debt when they claim their redemption

    }
```
