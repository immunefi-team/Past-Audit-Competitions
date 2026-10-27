# 30886 - \[SC - Medium] Wrong totalWeight in Votersol

Submitted on May 7th 2024 at 18:39:32 UTC by @cryptoticky for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30886

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

If the user proceeds with the vote only once and does nothing afterwards, the voting power of the user does not decrease and this affects the overall voting result.

## Vulnerability Details

```
/// @inheritdoc IVoter
    function notifyRewardAmount(uint256 amount) external {
        require(msg.sender == minter, "only minter can send rewards");
        require(totalWeight > 0, "no votes");

        _safeTransferFrom(base, msg.sender, address(this), amount); // transfer rewards in

        uint256 _ratio = (amount * 1e18) / totalWeight; // 1e18 adjustment is removed during claim

        if (_ratio > 0) {
            index += _ratio;
        }

        emit NotifyReward(msg.sender, base, amount);
    }
```

```
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

In this protocol, votingPower linearly decreases over time. However, if no action for the tokenId is taken after the first vote, the votingPower of that token remains within totalWeight and weights. This directly affects \_ratio within the Voter.notifyRewardAmount function. It also impacts claimable in the Voter.\_updateFor function.

When the Voter.distribute function is called, newly minted ALCX tokens are distributed to each gauge according to the voting results.

Ultimately, the voting results differ from what was intended, which affects the distribution of ALCX tokens.

Even if a token expires, this phenomenon continues.

## Impact Details

The governance voting results are manipulated, leading to a direct deviation from the intended impact of the original results.

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

    function testBugTotalWeight() public {
        address[] memory poolsOfAdmin = new address[](1);
        poolsOfAdmin[0] = sushiPoolAddress;
        address[] memory poolsOfBeef = new address[](1);
        poolsOfBeef[0] = balancerPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 10_000;


        uint256 nonActiveTokenId = createVeAlcx(admin, TOKEN_1 * 1000, MAXTIME, false);

        // go epoch 1
        goNextEpoch();

        // To call Minter.updatePeriod(). but not send reward token to Voter
        voter.distribute();


        hevm.prank(admin);
        voter.vote(nonActiveTokenId, poolsOfAdmin, weights, 0);

        uint256 totalWeight0 = voter.totalWeight();

        hevm.warp(block.timestamp + MAXTIME / 2);
        voter.distribute();


        uint256 totalWeight1 = voter.totalWeight();

        // ==================== General voting =======================
        // if user does nothing, Voter.totalWeight is not changed.
        assertEq(totalWeight0, totalWeight1, "The code is correct");

        uint256 outPutTokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        uint256 userTokenId = createVeAlcx(beef, TOKEN_1, MAXTIME, false);

        hevm.prank(admin);
        voter.vote(outPutTokenId, poolsOfAdmin, weights, 0);

        hevm.prank(beef);
        voter.vote(userTokenId, poolsOfBeef, weights, 0);

        uint256 period = minter.activePeriod();
        hevm.warp(period + nextEpoch);
        voter.distribute();

        address[] memory gauges = new address[](2);
        gauges[0] = address(sushiGauge);
        gauges[1] = address(balancerGauge);

        voter.updateFor(gauges);

        // be expected claimable amount for sushiGauge = claimable amount for balancerGauge
        uint256 claimableForSushi = voter.claimable(address(sushiGauge));
        uint256 claimableForBalancer = voter.claimable(address(balancerGauge));
        console.log(claimableForSushi);
        console.log(claimableForBalancer);

        // claimableForSushi > claimableForBalancer
        assertGt(claimableForSushi, claimableForBalancer, "claimable amount is equal");
        console.log("compare claimable: ", claimableForSushi / claimableForBalancer);
        // that shows 944
        // ================================================================

        period = minter.activePeriod();
        hevm.warp(period + nextEpoch);

        hevm.prank(admin);
        voter.reset(outPutTokenId);

        hevm.prank(beef);
        voter.reset(userTokenId);

        totalWeight1 = voter.totalWeight();
        assertEq(totalWeight0, totalWeight1, "totalWeight0 != totalWeight2");

        epochTime = epochTime + MAXTIME + DURATION;
        hevm.warp(epochTime);
        // at this time, the nonActiveTokenId is expired
        // but the totalWeight is kept the votingPower of the token when the token is created
        // the general vote will be continue ... and the expired votingPower affects the token distribution for gauges.
        // this TOKEN will have to be locked during MAXTIME at this point in order to get such a voting powers,
        // but the user can always withdraw at any time.
        totalWeight1 = voter.totalWeight();
        assertEq(totalWeight0, totalWeight1, "totalWeight0 != totalWeightAtExpiredTime");
    }
}
```
