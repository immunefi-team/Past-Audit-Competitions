# 31234 - \[SC - Medium] Alchemix BlockSlope variable in checkpoint rou...

Submitted on May 15th 2024 at 15:56:44 UTC by @Norah for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31234

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

* In the `_checkpoint()` routine of the **voterEscrow contract,** Global Point (pointHistory\[epoch]) for all previous epochs is updated within a for loop.
* This loop begins with the timestamp of the last Global checkpoint and increments by weeks with each iteration.
* The global point is updated at corresponding times with appropriate bias and slope in each iteration.
* For the block number, it is extrapolated on a line with the current time and block number as endpoints, using the last recorded values in pointHistory\[epoch] as the starting point.

## Vulnerability Details

* The vulnerability lies in the calculation of this extrapolation. Initially, the slope is calculated, and then this slope is used to determine the appropriate block number for any intermediate time.

```
      if (block.timestamp > lastPoint.ts) {
           blockSlope = (MULTIPLIER * (block.number - lastPoint.blk)) / (block.timestamp - lastPoint.ts);
       }

      uint256 _time = (lastCheckpoint / WEEK) * WEEK;
           
      for (uint256 i = 0; i < 255; ++i) {
            _time += WEEK;
                     .
                     .
                     .
         
              lastPoint.ts = _time;
              lastPoint.blk = initialLastPoint.blk + (blockSlope * (_time - initialLastPoint.ts)) / MULTIPLIER;
     

```

* The issue arises because most blockchains, including Ethereum, have a block rate of less than `1/2` , meaning it takes more than 2 seconds to confirm blocks.
* As a result, the **blockSlope will round down to zero, particularly with a low multiplier, such as 2.**
* This renders the scaling ineffective in preventing the rounding down to zero, especially when the block rate exceeds `1/2`.
* For example, ethereum has a blockrate of `1/12`; for this, it will be definitely rounded to zero.

## Impact Details

* The vulnerability results in incorrect block numbers being assigned, especially the last recorded block number of the global point, which will be assigned to all the the new points being updated.

## References

* Update the `constant MULTIPLIER` to higher value. (i.e >10000).

## References

Add any relevant links to documentation or code

## Proof of Concept

* Following POC demonstrates for block rate of `1/12`, how in current calculation `blockSlope` will evaluate to zero.
* To run the test, add the test in file in the existing test suite and then execute it with the command :
* "forge test --fork-url <https://eth-mainnet.g.alchemy.com/v2/{Alchemy-Key}> --match-test "testBlockSlope" -vv"
* check attachment for the output.

```solidity


function testBlockSlope() public {

        uint MULTIPLIER = 2;
        uint blockSlope; 

        uint lastPoint_ts = block.timestamp;
        uint lastPoint_blk = block.number;

        console2.log("lastPoint Timestamp   : ",block.timestamp);
        console2.log("lastPoint BlockNumber : ",block.number);

        uint duration = 2 weeks; //duration after wich the _checkpoint() routine is called.

        vm.warp(block.timestamp + duration);
        vm.roll(block.number + (duration)/12); // as per current block rate of 12 seconds a block.

        console2.log("Current Timestmap   : ",block.timestamp);
        console2.log("Current BlockNumber : ",block.number);
        

        if (block.timestamp > lastPoint_ts) {
                blockSlope = (MULTIPLIER * (block.number - lastPoint_blk)) / (block.timestamp - lastPoint_ts);
        }

        assertEq(blockSlope,0);

        //This will be rounded to zero.
        console2.log(blockSlope);
    }


```
