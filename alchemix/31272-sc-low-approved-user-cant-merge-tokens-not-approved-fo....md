# 31272 - \[SC - Low] Approved user cant merge tokens not approved fo...

Submitted on May 16th 2024 at 01:14:43 UTC by @OxAlix2 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31272

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

To merge 2 tokens into 2, a user must be either approved or owner of both tokens. This is obvious in the following checks in `VotingEscrow::merge`:

```
require(_isApprovedOrOwner(msg.sender, _from), "not approved or owner");
require(_isApprovedOrOwner(msg.sender, _to), "not approved or owner");
```

It also calls `_burn` which clears the approval of the "from" token, however, it's clearing it wrong as it calls `approve(address(0), _tokenId);`, which checks if the caller is the owner or approved for all, it doesn't allow "regular" approved users (which makes sense).

## Vulnerability Details

This blocks approved users (not for all) from merging 2 tokens, as the TX will revert in `approve`, which is not intended. The protocol should use `_clearApproval(owner, _tokenId);` instead.

## Impact Details

Approved users (not for all) aren't able to merge tokens.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L1567>

## Proof of Concept

```
function testApprovedCantMerge() public {
    uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
    uint256 tokenId2 = createVeAlcx(beef, TOKEN_100K, MAXTIME / 2, false);

    hevm.prank(admin);
    veALCX.approve(beef, tokenId1);

    assertEq(veALCX.getApproved(tokenId1), beef);
    assertEq(veALCX.ownerOf(tokenId2), beef);

    hevm.prank(beef);
    vm.expectRevert(abi.encodePacked("sender is not owner or approved"));
    veALCX.merge(tokenId1, tokenId2);
}
```
