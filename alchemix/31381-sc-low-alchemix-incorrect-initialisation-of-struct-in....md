# 31381 - \[SC - Low] Alchemix Incorrect Initialisation of struct in...

Submitted on May 17th 2024 at 19:51:58 UTC by @Norah for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31381

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

* In the `_checkpoint()` routine of the `voterEscrow` contract, the Global Point (pointHistory\[epoch]) for all previous epochs is updated within a for loop.
* For this, a memory struct called `lastPoint` is initialised with the last global point or with zero values of bias and slope, along with the latest timestamp and block number, in case it is called in the first epoch.
* Another struct, `initialLastPoint`, is also initialised by directly assigning the previously declared lastPoint

```solidity

        Point memory lastPoint = Point({ bias: 0, slope: 0, ts: block.timestamp, blk: block.number });

        if (_epoch > 0) {
            lastPoint = pointHistory[_epoch];
        }

        uint256 lastCheckpoint = lastPoint.ts;

        Point memory initialLastPoint = lastPoint;

```

* This loop begins with the timestamp of the last Global checkpoint and increments by weeks with each iteration.
* The global point is updated at corresponding times with appropriate bias and slope in each iteration.
* For the block number, it is extrapolated on a line with the current time and block number as endpoints, using the last recorded values in `initialLastPoint` as the starting point.

```solidity

        lastPoint.ts = _time;
        lastPoint.blk = initialLastPoint.blk + (blockSlope * (_time - initialLastPoint.ts)) / MULTIPLIER;

```

## Vulnerability Details

* The problem here is that `initialLastPoint` **and** `lastPoint` **both are pointing to the same struct in memory, instead of initialLastPoint pointing to a separate copy of it.**
* As a result, the entire calculation for updating the block number becomes incorrect.

```solidity

        lastPoint.blk = initialLastPoint.blk + (blockSlope * (_time - initialLastPoint.ts)) / MULTIPLIER;
```

* Since the values of \_time and initialLastPoint.ts will be the same, as \_time is read from lastPoint, which again points to the same struct as initialLastPoint.
* **Therefor, in each iteration, lastPoint.blk will evaluate to the same initialLastPoint.blk.** (as assigned initially).

## Impact Details

* The vulnerability results in incorrect block numbers being assigned, specifically the last recorded blockNnumber of the global point, will be assigned to all the the new points being updated.
* All the functionalities relaying on the totalVoting voter power at particular block.number will be catch incorrect values.

## Recomendation

* Initialized a new struct for the initialLastPoint in parallel with lastPoint instead of assigning it.

```solidity
        Point memory lastPoint = Point({ bias: 0, slope: 0, ts: block.timestamp, blk: block.number });
+++     Point memory initialLastPoint = IVeTetu.Point({bias: 0, slope: 0, ts: block.timestamp, blk: block.number});;
        
        if (_epoch > 0) {
            lastPoint = pointHistory[_epoch];
+++         initialLastPoint = pointHistory[_epoch];           
        }
        
    
---    Point memory initialLastPoint = lastPoint;
 
```

## Proof of Concept

* I have attached POC using the existing test suite show casing the impact of vulnerability.
* I have also attached additional test with simplified code of `_checkpoint()`, showcasing how both struct are pointing to the same struct instance in memory.
* To Run the tests add them (along with struct declaration) into the file `votingEscrow.t.sol` use command with following with respective names
* Run via command : "forge test --fork-url <https://eth-mainnet.g.alchemy.com/v2/{Alchemy> API Key} --match-test "testIncorrectBlockNumber" -vv"
* I have attached the screenshot of output of both the tests in the attachments.
* **Note :** Before running the test ensure that constant MULTIPLIER in the VotingEscrow is updated to 1e18,to avoid the impact due to another bug in the same functionality,(check Report #31234 for more detail).

```

    struct Point {
        int256 bias;
        int256 slope; 
        uint256 ts;  // time
        uint256 blk; // block
    }

    //Before running the test ensure that constant MULTIPLIER in the VotingEscrow is updated to 1e18
    //To avoid the impact due to another bug in the same functionality.
    //Check ReportID : 31234 for more detail.


    function testIncorrectBlockNumberWithSimplifiedCode() public {

        uint WEEK = 1 weeks;

        uint MULTIPLIER = 1e18;
        uint blockSlope = uint(MULTIPLIER/12); //standard block of rate of main-net

        Point memory lastPoint = Point({ bias: 0, slope: 0, ts: block.timestamp, blk: block.number });  
        uint lastCheckpoint =  lastPoint.ts;
        Point memory initialLastPoint = lastPoint;
        
        {
            uint256 _time = (lastCheckpoint / WEEK) * WEEK;

            for (uint256 i = 0; i < 5; ++i) {
                
                _time += WEEK;

                lastPoint.ts = _time;
                lastPoint.blk = initialLastPoint.blk + (blockSlope * (_time - initialLastPoint.ts)) / MULTIPLIER;

                //This will hold true since both structs are pointing to the same struct.
                assertEq(initialLastPoint.ts,lastPoint.ts);
                assertEq(initialLastPoint.blk,lastPoint.blk);

                console2.log("Iteration No         :  ", i);

                console2.log("lastPoint.ts         : ",lastPoint.ts);
                console2.log("initialLastPoint.ts  : ",initialLastPoint.ts);

                console2.log("lastPoint.blk        : ",lastPoint.blk);
                console2.log("initialLastPoint.blk : ",initialLastPoint.blk);

                console2.log("");
            }
        }
    }


    function testIncorrectBlockNumberOnVotingEscrow() public {

        //Declaring the Global Point
        Point memory GlobalPoint;

        //Creating a voter, here the first Global point will be recorded in call to checkPoint() called.
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, true);

        uint256 numCheckpoints = veALCX.numCheckpoints(admin);
        assertEq(numCheckpoints, 1, "numCheckpoints should be 1");

        (GlobalPoint.bias,,GlobalPoint.ts,GlobalPoint.blk) = veALCX.pointHistory(0);

        //Initial Block Numbers 
        console2.log("GlobalPoint values at Epoch : ", 0);
        console2.log("GlobalPoint BlockNumber :", GlobalPoint.blk);
        console2.log("GlobalPoint Timestamp   :", GlobalPoint.ts);
        console2.log("");

        //Now we move forward in time : 45 epochs.
        hevm.warp(block.timestamp + nextEpoch * 45);
        hevm.roll(block.number + nextEpoch * 45/12);

        //Calling the check point function after 45 epochs
        veALCX.checkpoint();

        //Ideally a block Number should have been updated as per the extrapolation formula.
        //But the intital block Number stays the same throughout.

        (GlobalPoint.bias,,GlobalPoint.ts,GlobalPoint.blk) = veALCX.pointHistory(45);
        console2.log("GlobalPoint values at Epoch : ", 45);
        console2.log("GlobalPoint BlockNumber :", GlobalPoint.blk);
        console2.log("GlobalPoint Timestamp   :", GlobalPoint.ts);
        console2.log("");

        (GlobalPoint.bias,,GlobalPoint.ts,GlobalPoint.blk) = veALCX.pointHistory(23);
        console2.log("GlobalPoint values at Epoch : ", 23);
        console2.log("GlobalPoint BlockNumber :", GlobalPoint.blk);
        console2.log("GlobalPoint Timestamp   :", GlobalPoint.ts);
        console2.log("");

        (GlobalPoint.bias,,GlobalPoint.ts,GlobalPoint.blk)= veALCX.pointHistory(8);
        console2.log("GlobalPoint values at Epoch : ", 8);
        console2.log("GlobalPoint BlockNumber :", GlobalPoint.blk);
        console2.log("GlobalPoint Timestamp   :", GlobalPoint.ts);
        console2.log("");

        (GlobalPoint.bias,,GlobalPoint.ts,GlobalPoint.blk)= veALCX.pointHistory(0);
        console2.log("GlobalPoint values at Epoch : ", 0);
        console2.log("GlobalPoint BlockNumber :", GlobalPoint.blk);
        console2.log("GlobalPoint Timestamp   :", GlobalPoint.ts);
        console2.log("");

        //Note : Also, You can run this test for the ReportID : 31234, after fixing this vulnerability and keeping 
        //the MULTIPLIER value same.

    }


```
