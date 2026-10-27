# 31418 - \[SC - Critical] the killed gauge collect claim amount

Submitted on May 18th 2024 at 21:46:26 UTC by @zeroK for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31418

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

the killed gauge can accrue claim amount even after it get killed, this is possible because the \_updateFor function won't prevent adding shares to killed gauge.

this report may get closed but we submit it as INSIGHTS for alchemix protocol

## Vulnerability Details

the \_updateFor did not prevent updating claimable amount for the killed gauge:

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
                claimable[_gauge] += _share;
            }
        } else {
            supplyIndex[_gauge] = index;
        }
    }


```

while the function is taken from the velo voter contract, the velo protocol set the claimable to zero when it kill it, but if this is not what the alchemix protocol want to do then the updatefor should not update the claimable amount for the killed gauge, in case if the gauge revived then the claimable amount can be huge amount till then.

## Impact Details

`updateFor` updates the claimable to killed gauge.

## Proof of Concept

```
    // SPDX-License-Identifier: GPL-3

// run forge test --match-test testExploit --fork-url <URL> -vv  -- > in src/test
pragma solidity ^0.8.15;

import "./BaseTest.sol";

import "lib/forge-std/src/console2.sol";
import "lib/forge-std/src/Test.sol";
import "./utils/DSTestPlus.sol";

import "src/interfaces/IRewardsDistributor.sol";

contract testing is Test, BaseTest {
    address public alice;
    uint256 internal constant THREE_WEEKS = 3 weeks;

    function setUp() public {
        setupContracts(block.timestamp);

        alice = vm.addr(1);
        deal(bpt, address(alice), TOKEN_100M);
    }

    function testExploit() public {
        //FIRST STEP get gauge address
        address emergencyCouncil = voter.emergencyCouncil();
        address gaugeAddress = voter.gauges(alEthPoolAddress);
        address[] memory guageS = new address[](1);
        guageS[0] = gaugeAddress;

        bool isGaugeAlive = voter.isAlive(gaugeAddress);
        assertEq(isGaugeAlive, true, "gauge should be alive");

        //SECOND STEP vote to the pool that point to the gauge

        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

        hevm.startPrank(admin);

        hevm.warp(block.timestamp + nextEpoch);

        address[] memory pools = new address[](1);
        pools[0] = alETHPool;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        voter.vote(tokenId, pools, weights, 0); // we vote to update weights of the gauge/pool

        uint256 period = minter.activePeriod();

        hevm.warp(period + nextEpoch);
        hevm.roll(block.number + 1);

        voter.distribute();

        // THIRD STEP kill the gauge

        hevm.startPrank(emergencyCouncil);
        voter.killGauge(gaugeAddress);

        hevm.stopPrank();

        // after killing the update for won't revert for killed gauge

        voter.updateFor(guageS);
    }
}

```
