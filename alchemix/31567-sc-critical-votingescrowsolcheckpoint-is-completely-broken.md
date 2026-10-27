# 31567 - \[SC - Critical] VotingEscrowsolcheckpoint is completely broken

Submitted on May 21st 2024 at 12:28:04 UTC by @gladiator111 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31567

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

`VotingEscrow.sol::checkpoint` is completely broken because of incorrect update of Epoch.

## Vulnerability Details

`Note - I have selected the closest impact. Please adjust the impact/severity as you may seem proper. Although it is clearly apparent but still I will present more POC/ evidence as to why this is a critical issue. As the time of boost is about to end, I am submitting this just confirming the basic vulnerability`

`Note for Immunefi Triage - Do not close this issue as the boost period is about to end. This is a legitimate issue, please read the whole report carefully, if any doubt ask in the comments. Thanks!`

In the `VotingEscrow.sol::_checkpoint`, the epoch is updated after every 1 week increment and not 2 week increment. So , the epochs are not updated correctly.

```solidity
function _checkpoint(uint256 _tokenId, LockedBalance memory oldLocked, LockedBalance memory newLocked) internal {
        Point memory oldPoint;
        Point memory newPoint;
        int256 oldDslope = 0;
        int256 newDslope = 0;
        uint256 _epoch = epoch;

        if (oldLocked.maxLockEnabled) oldLocked.end = ((block.timestamp + MAXTIME) / WEEK) * WEEK;
        if (newLocked.maxLockEnabled) newLocked.end = ((block.timestamp + MAXTIME) / WEEK) * WEEK;

        if (_tokenId != 0) {
            oldPoint = _calculatePoint(oldLocked, block.timestamp);
            newPoint = _calculatePoint(newLocked, block.timestamp);

            // Read values of scheduled changes in the slope
            // oldLocked.end can be in the past and in the future
            // newLocked.end can ONLY by in the FUTURE unless everything expired: then zeros
            oldDslope = slopeChanges[oldLocked.end];

            if (newLocked.end != 0) {
                if (newLocked.end == oldLocked.end) {
                    newDslope = oldDslope;
                } else {
                    newDslope = slopeChanges[newLocked.end];
                }
            }
        }

        Point memory lastPoint = Point({ bias: 0, slope: 0, ts: block.timestamp, blk: block.number });
        if (_epoch > 0) {
            lastPoint = pointHistory[_epoch];
        }
        uint256 lastCheckpoint = lastPoint.ts;
        // initialLastPoint is used for extrapolation to calculate block number
        // (approximately, for *At methods) and save them
        // as we cannot figure that out exactly from inside the contract
        Point memory initialLastPoint = lastPoint;
        uint256 blockSlope = 0; // dblock/dt
        if (block.timestamp > lastPoint.ts) {
            blockSlope = (MULTIPLIER * (block.number - lastPoint.blk)) / (block.timestamp - lastPoint.ts);
        }
        // If last point is already recorded in this block, slope=0
        // We know the block in such case

        // Go over weeks to fill history and calculate what the current point is
        {
            uint256 _time = (lastCheckpoint / WEEK) * WEEK;
            for (uint256 i = 0; i < 255; ++i) {
                // Hopefully it won't happen that this won't get used in 5 years!
                // If it does, users will be able to withdraw but vote weight will be broken
 @>             _time += WEEK;          // time increase by 1 week
                int256 dSlope = 0;
                if (_time > block.timestamp) {
                    _time = block.timestamp;
                } else {
                    dSlope = slopeChanges[_time];
                }
                int256 biasCalculation = lastPoint.slope * (int256(_time - lastCheckpoint));
                // Make sure we still subtract from bias if value is negative
                biasCalculation >= 0 ? lastPoint.bias -= biasCalculation : lastPoint.bias += biasCalculation;
                lastPoint.slope += dSlope;
                if (lastPoint.bias < 0) {
                    // This can happen
                    lastPoint.bias = 0;
                }
                if (lastPoint.slope < 0) {
                    // This cannot happen - just in case
                    lastPoint.slope = 0;
                }
                lastCheckpoint = _time;
                lastPoint.ts = _time;
                lastPoint.blk = initialLastPoint.blk + (blockSlope * (_time - initialLastPoint.ts)) / MULTIPLIER;
@>              _epoch += 1;         // epoch increases by 1 (i.e 2 weeks)
                if (_time == block.timestamp) {
                    lastPoint.blk = block.number;
                    break;
                } else {
                    pointHistory[_epoch] = lastPoint;
                }
            }
        }

        epoch = _epoch;
        // Now pointHistory is filled until t=now

        if (_tokenId != 0) {
            // If last point was in this block, the slope change has been applied already
            // But in such case we have 0 slope(s)

            lastPoint.slope += (newPoint.slope - oldPoint.slope);
            lastPoint.bias += (newPoint.bias - oldPoint.bias);

            if (lastPoint.slope < 0) {
                lastPoint.slope = 0;
            }
            if (lastPoint.bias < 0) {
                lastPoint.bias = 0;
            }
        }

        // Record the changed point into history
        pointHistory[_epoch] = lastPoint;

        if (_tokenId != 0) {
            // Schedule the slope changes (slope is going down)
            // We subtract from [newLocked.end]
            // and add to [oldLocked.end]
            if (oldLocked.end > block.timestamp) {
                // oldDslope was <something> - oldPoint.slope, so we cancel that
                oldDslope += oldPoint.slope;
                if (newLocked.end == oldLocked.end) {
                    oldDslope -= newPoint.slope; // It was a new deposit, not extension
                }
                slopeChanges[oldLocked.end] = oldDslope;
            }

            if (newLocked.end > block.timestamp) {
                if (newLocked.end > oldLocked.end) {
                    newDslope -= newPoint.slope; // oldPoint slope disappeared at this point
                    slopeChanges[newLocked.end] = newDslope;
                }
                // else: we recorded it already in oldDslope
            }
            // Handle user history
            uint256 userEpoch = userPointEpoch[_tokenId] + 1;

            userPointEpoch[_tokenId] = userEpoch;
            newPoint.ts = block.timestamp;
            newPoint.blk = block.number;
            userPointHistory[_tokenId][userEpoch] = newPoint;
        }
    }
```

This leads to the whole checkpoint function being broken. This checkpoint function is crucial for all parts of the protocols. So, if this breaks then the whole protocol, including distributing the rewards etc breaks. For Example:- This checkpoint function is also used in `RewardDistributor::_checkpointTotalSupply()`

```solidity
function _checkpointTotalSupply() internal {
        address ve = votingEscrow;
        uint256 t = timeCursor;
        uint256 roundedTimestamp = (block.timestamp / WEEK) * WEEK;
@>      IVotingEscrow(ve).checkpoint();   //This function is also used here

        for (uint256 i = 0; i < 20; i++) {
            if (t > roundedTimestamp) {   //if timeCursor falls within the current epoch then just let it as it is
                break;
            } else {
                veSupply[t] = IVotingEscrow(ve).totalSupplyAtT(t);
            }
            t += WEEK;
        }
        timeCursor = t;
    }
```

This effectively breaks the entire protocol.

## Impact Details

The whole functionality of the protocol breaks

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1170-1301>

## Proof of Concept

This POC shows that after passing just 1 EPOCH timestamp, the epoch gets from 0 to 4 clearly indicating broken epoch accounting. Paste the following code in `VotingEscrow.t.sol` and run using the command

```bash
forge test --match-test testEpochBreaks -vvvv --fork-url $FORK_URL
```

```solidity
    function testEpochBreaks() public {
        veALCX.checkpoint();                                // checkpointing
        console.log(veALCX.epoch());                        // outputs 1
        vm.warp(block.timestamp + nextEpoch );              // Forwarding 1 Epoch
        veALCX.checkpoint();                                //checkpointing
        console.log(veALCX.epoch());                        // outputs 4 despite only one Epoch forwarding
    }
```
