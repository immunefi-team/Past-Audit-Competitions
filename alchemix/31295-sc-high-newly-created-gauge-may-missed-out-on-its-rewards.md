# 31295 - \[SC - High] Newly created gauge may missed out on its rewards

Submitted on May 16th 2024 at 17:25:04 UTC by @Lin511 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31295

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Newly created gauge may missed out on its rewards when the first distribute took place, due to the incorrect use of memory variables.

## Vulnerability Details

In Voter.\_distribute(), `claimable[_gauge]` is assigned to a memory variable `_claimable`, then `claimable[_gauge]` is reset to zero.

```solidity
    function _distribute(address _gauge) internal {
        // Distribute once after epoch has ended
        require(
            block.timestamp >= IMinter(minter).activePeriod() + IMinter(minter).DURATION(),
            "can only distribute after period end"
        );

        uint256 _claimable = claimable[_gauge];

        // Reset claimable amount
        claimable[_gauge] = 0;

        _updateFor(_gauge);

        if (_claimable > 0) {
            IBaseGauge(_gauge).notifyRewardAmount(_claimable);
        }

        ...
    }
```

After that `claimable[_gauge]` is updated in \_updateFor(\_gauge).

```solidity
    function _updateFor(address _gauge) internal {
        require(isGauge[_gauge], "invalid gauge");

        address _pool = poolForGauge[_gauge];
        uint256 _supplied = weights[_pool];
        if (_supplied > 0) {
            uint256 _supplyIndex = supplyIndex[_gauge];
            uint256 _index = index; // get global index0 for accumulated distro
            supplyIndex[_gauge] = _index; // update _gauge current position to global position
            uint256 _delta = _index - _supplyIndex; // see if there is any difference that need to be accrued
            if (_delta > 0) {
                uint256 _share = (uint256(_supplied) * _delta) / 1e18; // add accrued difference for each supplied token
@>                claimable[_gauge] += _share;
            }
        } else {
            supplyIndex[_gauge] = index;
        }
    }

```

At last, if `_claimable` is greater than zero, reward will be send to gauge.\
There is a problem with that `_claimable` is a memory variable, when `claimable[_gauge]` be updated, `_claimable` is not along with it, so there is a scenario that gauge won't receive it's reward on it's first distribute:\
1, `createGauge()` called, gauge A created.\
2, some one vote for it.\
3, minter called `notifyRewardAmount()`.\
4, some one call `distribute()`, it's the first distribute of gauge A, when contracts runs to `distribute(guage A)`, `claimable[_gauge]` is greater than zero but `_claimable` is equal to zero, so gauge A missed out on it's reward this time.

## Impact Details

Contracts may not work as intended, in the worst-case scenario, if the distribute() function is called only once, newly created guage could lose its rewards.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L366-L375> <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L481>

## Proof of Concept

```solidity
// SPDX-License-Identifier: GPL-3
pragma solidity ^0.8.15;

import "./BaseTest.sol";
import "forge-std/console.sol";

contract VotingTest2 is BaseTest {
    function setUp() public {
        setupContracts(block.timestamp);
    }

    function testNewlyCreatedGauge() public {

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;
        hevm.prank(voter.admin());
        // Gauge A created.
        console.log("Gauge A created.");
        address gauge = voter.createGauge(sushiPoolAddress, IVoter.GaugeType.Passthrough);

        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        hevm.prank(admin);
        // Some one vote for the pool related to gauge A.
        console.log("Some one vote for the pool related to gauge A.");
        voter.vote(tokenId, pools, weights, 0);
        assertEq(alcx.balanceOf(gauge), 0);

        // Give minter some ethers.
        deal(address(alcx), address(minter), TOKEN_100K);

        hevm.startPrank(address(minter));
        require(alcx.approve(address(voter), TOKEN_100K), 'approve failed');
        // Minter call notifyRewardAmount().
        console.log("Minter call notifyRewardAmount().");
        voter.notifyRewardAmount(TOKEN_100K);
        hevm.stopPrank();

        // First distribute of gauge A, no rewards received.
        // sushiPoolAddress is the reward receiver of gauge A, so we just need to monitor it's alcx balance.
        console.log("First distribute of gauge A, no rewards received.");
        uint256 balanceBefore = alcx.balanceOf(sushiPoolAddress);
        hevm.warp(minter.activePeriod() + minter.DURATION());
        voter.distribute();
        uint256 balanceAfter = alcx.balanceOf(sushiPoolAddress);
        assertEq(balanceBefore, balanceAfter);

        // Only by distribute again, gauge A can receive its rewards.
        console.log("Only by distribute again, gauge A can receive its rewards.");
        balanceBefore = alcx.balanceOf(sushiPoolAddress);
        hevm.warp(minter.activePeriod() + minter.DURATION() + minter.DURATION());
        voter.distribute();
        balanceAfter = alcx.balanceOf(sushiPoolAddress);
        assertGt(balanceAfter, balanceBefore);
    }

}
```
