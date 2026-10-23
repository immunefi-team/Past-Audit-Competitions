# 31087 - \[SC - Low] Colition between approve and \_isApprovedOrOwner...

Submitted on May 12th 2024 at 14:40:44 UTC by @Ch301 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31087

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Users with `approve()` can't trigger `merge()` function.

## Vulnerability Details

When a user (has the approve) triggers `VotingEscrow.sol#merge()` the `_burn()` function will sub-call to `approve()`

```solidity
File: VotingEscrow.sol

1601:     function _burn(uint256 _tokenId, uint256 _value) internal {
             /***/
1609:         // Clear approval
1610:         approve(address(0), _tokenId);

```

However, the `approve()` will revert if: `msg.sender` is not the owner and `(ownerToOperators[owner])[msg.sender]` returns false.

```solidity
File: VotingEscrow.sol
501:     function approve(address _approved, uint256 _tokenId) public {
        /***/
507:         // Check requirements
508:         bool senderIsOwner = (owner == msg.sender);
509:         bool senderIsApprovedForAll = (ownerToOperators[owner])[msg.sender];
510: 
511:         require(senderIsOwner || senderIsApprovedForAll, "sender is not owner or approved");
```

## Impact Details

The owner sets both the NFTs `approve()` to the user. however, he cannot call `merge()` successfully.

## References

non

## Proof of Concept

Foundry PoC:

1. Please copy the following POC in `VotingEscrow.t.sol`

```solidity
 function test_poc() public {
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(admin, TOKEN_100K, MAXTIME / 2, false);

        hevm.startPrank(admin);
        // Vote to trigger flux accrual
        hevm.warp(newEpoch());

        address[] memory pools = new address[](1);
        pools[0] = alETHPool;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;
        voter.vote(tokenId1, pools, weights, 0);
        voter.vote(tokenId2, pools, weights, 0);

        voter.distribute();

        hevm.warp(newEpoch());

        // Reset to allow merging of tokens
        voter.reset(tokenId1);
        //voter.reset(tokenId2); No needs
        veALCX.approve(beef, tokenId1);
        veALCX.approve(beef, tokenId2);
        hevm.stopPrank();

        hevm.prank(beef);
        hevm.expectRevert(abi.encodePacked("sender is not owner or approved"));
        veALCX.merge(tokenId1, tokenId2);
    }
```

2. Test result:

```diff
Ran 1 test for src/test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] test_poc() (gas: 4059754)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 93.81s (68.85s CPU time)

Ran 1 test suite in 96.06s (93.81s CPU time): 1 tests passed, 0 failed, 0 skipped 
(1 total tests)
```
