# 31425 - \[SC - Medium] Users can call reset on their token even if the...

Submitted on May 18th 2024 at 23:01:57 UTC by @OxAlix2 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31425

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Users can vote on certain gauges in the `Voter` contract, allowing them to "withdraw" their votes using the `reset` function, a note on this function, is that it sets `lastVoted` for that token ID. However, that function doesn't check if the token has active votes before allowing the user to reset them. This introduces an issue, especially when users move/transfer/sell their tokens, more below\...

## Vulnerability Details

A bit of context, in `VotingEscrow::_transferFrom`, there's a check to block the transferring of tokens in case that token has active votes.

```
require(!voted[_tokenId], "voting in progress for token");
```

This is not there to block users from voting multiple times using the same token, as the vote is saved per token and not per owner. So, this is there to allow a token receiver to immediately have the ability to use that token and vote on gauges.

So a user can use the missing check anomaly to grief NFT buyers (remember tokens are NFTs and are tradable), so a user puts his token for sale, and just before the transferal that user front runs the transfer TX and calls `Voter::reset`, this blocks the buyer from voting on any gauge in the current epoch.

## Impact Details

Users can grief token buyers/receivers and block them from voting in the current epoch.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L183-L192>

## Mitigation

Add the following in `Voter::reset`:

```
require(poolVote[_tokenId].length > 0, "No active votes");
```

This could be added in `if (msg.sender != admin)` just in case the admin wanted a higher authority in case of an error.

## Proof of concept

```
function testBlockVotingForBuyers() public {
    // Admin creates lock
    uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

    // Admin calls reset on his own token
    vm.prank(admin);
    voter.reset(tokenId);

    // token's `lastVoted` is now, this is wrong as there's no active vote
    assertEq(voter.lastVoted(tokenId), block.timestamp);

    // Admin sends/sells the token to beef
    vm.prank(admin);
    veALCX.safeTransferFrom(admin, beef, tokenId);

    address[] memory pools = new address[](1);
    pools[0] = alUsdPoolAddress;
    uint256[] memory weights = new uint256[](1);
    weights[0] = 5000;

    // Beef votes revert as he can't vote in this epoch (last voted was updated in `reset`)
    vm.prank(beef);
    vm.expectRevert(abi.encodePacked("TOKEN_ALREADY_VOTED_THIS_EPOCH"));
    voter.vote(tokenId, pools, weights, 0);
}
```
