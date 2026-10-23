# 31566 - \[SC - Medium] Checkpoints wont update block number in point b...

Submitted on May 21st 2024 at 12:27:19 UTC by @copperscrewer for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31566

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Voting Escrow Checkpoints won't update block number in point because of a Rounding issue, currently the block time in Ethereum is around \~ 12 seconds, There is an attempt to calculate block slope by the difference of the block number multiplied by a MULTIPLIER and then divided by the difference in block.timestamp.

There is an issue because of the precision of the division.

## Vulnerability Details

In \_checkpoint of VotingEscrow there is a line

`blockSlope = (MULTIPLIER * (block.number - lastPoint.blk)) / (block.timestamp - lastPoint.ts);`

Here blockSlope will always be zero as for each block.number the difference in block.timestamp is around \~ 12 seconds. The MULTIPLIER : 2 is too low to cover that and as a result it becomes zero as the numerator is lesser than the denominator

`lastPoint.blk = initialLastPoint.blk + (blockSlope * (_time - initialLastPoint.ts)) / MULTIPLIER;` as a result will be stagnant and never be updated.

## Impact Details

The Point objects used in \_checkpoint never have their block value updated. This will affect any off-calculation/computation that relies on the block number for points to make a transaction

## Remediation

Increase MULTIPLIER to a much higher value \~ 10000

## Proof of Concept

Run `forge test --match-test testRounding --fork-url https://eth-mainnet.alchemyapi.io/v2/[API_KETY] -vvv`

after pasting this function in VotingEscrow\.t.sol

```solidity
function testRounding() external
    {        

        VotingEscrowTest.Point memory lastPoint;
        (int256 a, int256 b, uint256 c, uint256 d) = veALCX.pointHistory(veALCX.epoch());
        lastPoint = Point(a,b,c,d);

        hevm.warp(block.timestamp + 12);
        hevm.roll(block.number + 1);
        
        console.log("block.timestamp : ", block.timestamp, "\tBlock.number : ", block.number);
        uint256 blockSlope = (MULTIPLIER * (block.number - lastPoint.blk)) / (block.timestamp - lastPoint.ts);
        console.log("blockSlope is ", blockSlope);

        veALCX.checkpoint();
        hevm.warp(block.timestamp + 12);
        hevm.roll(block.number + 1);

        ( a,  b,  c,  d) = veALCX.pointHistory(veALCX.epoch());
        lastPoint = Point(a,b,c,d);

        console.log("block.timestamp : ", block.timestamp, "\tBlock.number : ", block.number);
        blockSlope = (MULTIPLIER * (block.number - lastPoint.blk)) / (block.timestamp - lastPoint.ts);
        console.log("blockSlope is ", blockSlope);

        veALCX.checkpoint();

        hevm.warp(block.timestamp + 12*10);
        hevm.roll(block.number + 1*10);

        ( a,  b,  c,  d) = veALCX.pointHistory(veALCX.epoch());
        lastPoint = Point(a,b,c,d);
        
        console.log("block.timestamp : ", block.timestamp, "\tBlock.number : ", block.number);
        blockSlope = (MULTIPLIER * (block.number - lastPoint.blk)) / (block.timestamp - lastPoint.ts);
        console.log("blockSlope is ", blockSlope);

        veALCX.checkpoint();

    }
```
