# 30818 - \[SC - Low] division before multiplication in theamountToRa...

Submitted on May 6th 2024 at 19:37:35 UTC by @zeroK for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30818

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

when call made to the `startCooldown` function the `amountToRagequit` is called when the user want to withdraw before the end time reached, this function return the claimable amount flux for the user tokenId and the totalEpochs which is the total epochs in year multiplied by the fluxMultiplier, however the totalEpochs is not correct, because we do division before multiplication, this lead to return less ragequitAmount than the expected value.

## Vulnerability Details

the `amountToRagequit` is implemented as below:

```solidity

function amountToRagequit(uint256 _tokenId) public view returns (uint256) {
        // amount of flux earned in one epoch
        uint256 oneEpochFlux = claimableFlux(_tokenId);

        // total amount of epochs in fluxMultiplier amount of years
        uint256 totalEpochs = fluxMultiplier * ((MAXTIME) / EPOCH); //@audit we div before mul

        // based on one epoch, calculate total amount of flux over fluxMultiplier amount of years
        uint256 ragequitAmount = oneEpochFlux * totalEpochs;

        return ragequitAmount;
    }
```

the `totalEpochs` is not correct because we say maxtime / epoch then \* fluxMultiplier , in math, the amount and actions inside brackets is executed before any other actions and this cause rounding error in solidity which cause return less amount than expected(this is good for any malicious users who want to exist with less amount to burn)

## Impact Details

division before multiplication cause rounding error/

## Recommend

do multiplication before division.

## Proof of Concept

run the contract below in remix to show the difference between the value, one of them `MulandDiv` return 730 and the `divandMul` return 728.

```solidity
pragma solidity ^0.8.19;

 contract testDivAndMul {

    uint256 public week = 2 weeks; 
    uint256 public max_time = 365 weeks;

     uint256 public fluxMultiplier = 4 ;// 4x as mentioned

    function divandMul() public view returns(uint) {
        return fluxMultiplier * ((max_time) / week);
    } 

    function MulandDiv() public view returns(uint) {
        return (fluxMultiplier * max_time) / week;
    } 

    
}
```
