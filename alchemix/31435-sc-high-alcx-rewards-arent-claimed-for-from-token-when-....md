# 31435 - \[SC - High] ALCX rewards arent claimed for from token when ...

Submitted on May 19th 2024 at 02:47:23 UTC by @OxAlix2 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31435

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Permanent freezing of unclaimed yield
* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In `VotingEscrow::withdraw`, the protocol is claiming the ALCX rewards before burning the token, which makes sense as the token will be burnt. However, this is not done when merging 2 tokens, this puts the ALCX rewards of the "from" token at risk of being stuck forever.

## Vulnerability Details

When merging 2 tokens, token\_1, and token\_2, assume that token\_1 has some unclaimed ALCX rewards, through the merge process token\_1 will be burnt. So all these unclaimed ALCX will remain stuck forever as `RewardsDistributor::claim` will revert on the following:

```
require(approvedOrOwner || isVotingEscrow, "not approved");
```

because the token doesn't exist anymore.

## Impact Details

ALCX rewards that were accumulated for the "from" token will remain unclaimable/stuck forever after the merge process.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L618-L651>

## Mitigation

Add the following in `VotingEscrow::merge`:

```
IRewardsDistributor(distributor).claim(_from, false);
```

## Proof of concept

```
function testALCXRewardsNotClaimedOnMerge() public {
    uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
    uint256 tokenId2 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

    hevm.warp(minter.activePeriod() + nextEpoch);
    voter.distribute();

    assertGt(distributor.claimable(tokenId1), 0);
    assertGt(distributor.claimable(tokenId2), 0);

    hevm.prank(admin);
    veALCX.merge(tokenId1, tokenId2);

    assertGt(distributor.claimable(tokenId1), 0);
    assertGt(distributor.claimable(tokenId2), 0);

    vm.prank(admin);
    vm.expectRevert(abi.encodePacked("not approved"));
    distributor.claim(tokenId1, false);
}
```
