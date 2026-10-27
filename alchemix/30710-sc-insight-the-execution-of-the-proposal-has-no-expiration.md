# 30710 - \[SC - Insight] The execution of the proposal has no expiration

Submitted on May 5th 2024 at 13:17:09 UTC by @OxG0P1 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30710

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

If successful, any proposal can be executed at any time, as there is no expiration date for proposals.

## Vulnerability Details

After successful completion, a proposal will be executed following a specified delay through the `execute` function. However, a vulnerability arises from the fact that the function fails to verify whether the proposal has expired. In fact, there is no implementation of an expiration mechanism within the governance framework. This omission poses significant risks.

## Impact Details

Consider the following scenario:

Alice submits Proposal A to stake 20,000 ETH to a DEFI protocol, which successfully passes. However, it cannot be executed due to only 15,000 ETH remaining in the timelock, depleted by other proposals. Proposal A lacks an expiration period so it can be executed anytime. Subsequently, the DEFI protocol falls victim to a hack or rug-pull three days later. At this point, the Timelock accumulates sufficient funds to execute Proposal A.

Due to the absence of an expiration mechanism, Proposal A can be executed at any time, including after the protocol's compromise. Even if governance attempts to 'cancel' Proposal A, a malicious actor could front-run this transaction and execute Proposal A, leading to severe damage to the protocol.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/governance/L2Governor.sol#L352-L375>

## Proof of Concept

`Test :`

```solidity
    function testOnlyExecutorCanExecute() public {
        assertFalse(voter.isWhitelisted(usdc));

        (address[] memory t, uint256[] memory v, bytes[] memory c, string memory d) = craftTestProposal();

        hevm.warp(block.timestamp + 2 days); // delay

        // propose
        hevm.startPrank(admin);
        uint256 pid = governor.propose(t, v, c, d, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // voting delay
        hevm.roll(block.number + 1);
        hevm.stopPrank();

        // vote
        hevm.startPrank(admin);
        governor.castVote(pid, 1);
        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period
        hevm.stopPrank();

        // execute
        hevm.startPrank(admin);
        // execution delay
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); 


        hevm.warp(block.timestamp + 1000 weeks); //After 1000 weeks

        uint proposalId = governor.execute(t, v, c, keccak256(bytes(d)), MAINNET);
        assertEq(proposalId, pid);
    }
```

`Result :`

```solidity
Ran 1 test for src/test/AlchemixGovernor.t.sol:AlchemixGovernorTest
[PASS] testOnlyExecutorCanExecute() (gas: 345470)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 83.23s (695.80µs CPU time)

Ran 1 test suite in 85.45s (83.23s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
