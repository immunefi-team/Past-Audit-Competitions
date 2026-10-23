# 30910 - \[SC - High] Processing of voting results is not implemented...

Submitted on May 7th 2024 at 23:52:40 UTC by @cryptoticky for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30910

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

Processing of voting results is not implemented in the next epoch.

## Vulnerability Details

When `Voting.distribute` function is calling, the `Voting.notifyRewardAmount` is called at the end. It is also inconsistent in calls of the `_updateFor` function.

`_updateFor` function is called before other variables are updated in `Voter.vote` and `Voter.reset` functions. But in `Voter._distribute` function, the voter sends `alcx` token with old `claimable` value. So in the code, `Voter._updateFor` function is called before sending the `alcx` token but this produces the same result that this call is made at the end of the function.

## Impact Details

As a result, the voting result goes against what was expected, and the processing of each voting result has an epoch-sized delay.

However, I registered this report as medium because there is no actual loss of funds.

## Recommendation

1. Move `IMinter(minter).updatePeriod();` at the start of distribute function

```
/// @inheritdoc IVoter
    function distribute() external {
        IMinter(minter).updatePeriod();

        uint256 start = 0;
        uint256 finish = pools.length;

        for (uint256 x = start; x < finish; x++) {
            // We don't revert if gauge is not alive since pools.length is not reduced
            if (isAlive[gauges[pools[x]]]) {
                _distribute(gauges[pools[x]]);
            }
        }
    }
```

2. Update the `_distribute` function like this

```
function _distribute(address _gauge) internal {
        // Distribute once after epoch has ended
        require(
            block.timestamp >= IMinter(minter).activePeriod() + IMinter(minter).DURATION(),
            "can only distribute after period end"
        );

        _updateFor(_gauge);

        uint256 _claimable = claimable[_gauge];

        // Reset claimable amount
        claimable[_gauge] = 0;

        if (_claimable > 0) {
            IBaseGauge(_gauge).notifyRewardAmount(_claimable);
        }

        IBribe(bribes[_gauge]).resetVoting();

        emit DistributeReward(msg.sender, _gauge, _claimable);
    }
```

3. Don't call `_updateFor` function in other functions. `claimable` variable is only used in `_distribute` function and `distribute` function is called only one time in an epoch period. So don't need to update that variable in vote and reset function.

***

This is just a recommendation. You can find a better solution.

## Proof of Concept

```
// SPDX-License-Identifier: GPL-3
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract VoterPoC is BaseTest {
    uint256 constant DURATION = 2 weeks;
    uint256 constant SECONDS_PER_BLOCK = 12;
    uint256 public epochTime;
    uint256 public epochBlock;

    address public sushiBribeAddress;
    address public balancerBribeAddress;

    function setUp() public {
        setupContracts(block.timestamp);
        epochTime = minter.activePeriod();
        epochBlock = block.number;
    }

    function goNextEpoch() private {
        epochTime = epochTime + DURATION;
        epochBlock = epochBlock + DURATION / 12;
        hevm.warp(epochTime);
        hevm.roll(epochBlock);

        sushiBribeAddress = voter.bribes(address(sushiGauge));
        balancerBribeAddress = voter.bribes(address(balancerGauge));

    }

    function testBugDepositReward() public {
        address[] memory poolsOfAdmin = new address[](1);
        poolsOfAdmin[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 10_000;

        // go epoch 1
        uint256 period = minter.activePeriod();
        hevm.warp(period + nextEpoch);
        // Voter.totalWeight is 0
        // Voter.notifyRewardAmount is not called
        // Voter.index is 0
        voter.distribute();

        uint256 targetTokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        hevm.prank(admin);
        voter.vote(targetTokenId, poolsOfAdmin, weights, 0);

        uint256 claimableForSushi = voter.claimable(address(sushiGauge));
        // at this time, the claimable variable is 0
        assertEq(claimableForSushi, 0, "claimableForSushi > 0");

        // go epoch 2
        period = minter.activePeriod();
        hevm.warp(period + nextEpoch);

        // Calling Voter.distribute function in the new epoch, the reward token should be sent to gauges.
        // But no reward is sent to gauges
        // 1. call _distribute function
        // 2. call _updateFor function => claimable is 0 because delta index is still 0
        // 3. call Minter.updatePeriod function
        // 4. call Voter.notifyRewardAmount function because Voter.totalWeight is greater than 0.
        voter.distribute();
        claimableForSushi = voter.claimable(address(sushiGauge));
        // but the updateFor function is not called yet
        // at this time, the claimable variable is still 0.
        assertEq(claimableForSushi, 0, "claimableForSushi > 0");
        // Now the reward token is in Voter contract.
        assertGt(alcx.balanceOf(address(voter)), 0, "alcx balance of Voter contract is 0");
        // This means that processing of voting results is not done in the next epoch.

        // call reset function to avoid the problem pointed out in the report #30886
        hevm.prank(admin);
        // totalWeight -> 0 because of there is no vote.
        voter.reset(targetTokenId);

        // go epoch 3
        period = minter.activePeriod();
        hevm.warp(period + nextEpoch);

        voter.distribute();
        // Now the alcx token is sent to pool
        // Small amounts of alcx may remain due to rounding calculations.
        // In fact, 1 wei of alcx is left in this test.
        assertLt(alcx.balanceOf(address(voter)), 2, "alcx balance of Voter contract is not 0");
        // In the end, it can be seen that the processing of voting results in epoch 1 takes place only after epoch 3.

    }
}
```
