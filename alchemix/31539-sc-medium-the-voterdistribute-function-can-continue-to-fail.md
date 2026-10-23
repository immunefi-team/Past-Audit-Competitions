# 31539 - \[SC - Medium] The Voterdistribute function can continue to fail

Submitted on May 21st 2024 at 04:16:06 UTC by @cryptoticky for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31539

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Protocol insolvency
* Unbounded gas consumption

## Description

## Brief/Intro

When calling the Voter.distribute function, the distribution for some gauges can fail for a number of reasons and the transaction would be failed.

Therefore, it is not appropriate to distribute while circulating all the gauges. In addition, as the number of gauges increases, gas costs can exceed blockGasLimit in the worst case. This can cause the protocol to stop.

## Vulnerability Details

### 1. Failure due to some conditional statements

#### This is not about the gauge that the attacker artificially adds. This is a problem that can occur in already existing gates

```
/**
     * @notice Set the proposal id
     * @param _proposal Proposal id from snapshot url
     * @dev Proposal id must be set manually every epoch
     */
    function updateProposal(bytes32 _proposal) external {
        require(msg.sender == admin, "not admin");
        proposal = _proposal;
        proposalUpdated = true;

        emit ProposalUpdated(proposal, proposalUpdated);
    }

    /*
        Internal functions
    */

    /**
     * @notice Logic to pass rewards to votium contract
     * @param _amount Amount of rewards
     */
    function _passthroughRewards(uint256 _amount) internal override {
        require(initialized, "gauge must me initialized");
        require(proposalUpdated == true, "proposal must be updated");

        // Reset proposal flag
        proposalUpdated = false;

        bytes32 proposalHash = keccak256(abi.encodePacked(proposal));

        uint256 rewardBalance = IERC20(rewardToken).balanceOf(address(this));
        require(rewardBalance >= _amount, "insufficient rewards");

        IERC20(rewardToken).approve(receiver, _amount);
        IVotiumBribe(receiver).depositBribe(rewardToken, _amount, proposalHash, poolIndex);

        emit Passthrough(msg.sender, rewardToken, _amount, receiver);
    }
```

If proposalUpdated is false, the distribute function is failed. So the admin must call the updateProposal function before call distribution function at start time of every epoch. However, there may be situations where you should not update the proposal in some gauges Then, you will not be able to distribute all the gages because of this gauge.

### 2. Failure by operation beyond block gas limit

Treating too many gates into one loop may exceed blockGasLimit.

## Impact Details

The Voter.distribute function can continue to fail and the protocol may be stopped. If tx fails, the corresponding gas cost will be lost. If the protocol has 100 gauges and successfully run up to 99 but fail to run on the last 100th gauge, you lose significant gas costs.

## Recommendation

Add these functions in Voter.sol

```
function distribute(uint256 _start, uint256 _finish) external;
```

```
function distribute(address[] memory _gauges) external;
```

***

I want you to look at the problems in this report precisely and carefully. This is a problem that often happens in real life.

Thank you.

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

    function testFailedDistribute() public {
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        address bribeAddress = voter.bribes(address(alUsdGauge));
        address[] memory pools = new address[](1);
        pools[0] = alUsdPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        hevm.prank(admin);
        voter.vote(tokenId, pools, weights, 0);

        hevm.warp(newEpoch());
        voter.distribute();

        hevm.prank(admin);
        voter.vote(tokenId, pools, weights, 0);

        hevm.warp(newEpoch());
        hevm.expectRevert(abi.encodePacked("proposal must be updated"));
        voter.distribute();
    }
}
```
