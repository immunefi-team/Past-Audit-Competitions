# 31281 - \[SC - Low] Approved spender cannot withdraw or merge

Submitted on May 16th 2024 at 07:15:47 UTC by @OxAnmol for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31281

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Temporary freezing of NFTs

## Description

## Brief/Intro

Users who are approved, but do not own a particular NFT, are supposed to be eligible to call merge and withdraw from the NFT.

Currently, *`_burn()`, used by `merge()` and `withdraw()` to remove the NFT from the system, will revert unless the sender is the owner of the NFT as the public `approve` called inside* `_burn` requires the sender to be the owner or operator.

## Vulnerability Details

The merge function and withdraw function is calling internal `_burn`

```solidity
function merge(uint256 _from, uint256 _to) external {
        ...SNIP...
        _burn(_from, value0);
        _depositFor(_to, value0, end, _locked1.maxLockEnabled, _locked1, DepositType.MERGE_TYPE);
    }
```

Now if we have a look at `_burn` it calls the public `approve` function to set the token approval to `address(0)`

```solidity
  function _burn(uint256 _tokenId, uint256 _value) internal {
        address owner = ownerOf(_tokenId);

        // Update the total supply of deposited tokens
        uint256 supplyBefore = supply;
        uint256 supplyAfter = supplyBefore - _value;
        supply = supplyAfter;

        // Clear approval
        //@audit-issue This will revert for approved users
        approve(address(0), _tokenId);
        // Checkpoint for gov
        _moveTokenDelegates(delegates(owner), address(0), _tokenId);
        // Remove token
        _removeTokenFrom(owner, _tokenId);
        emit Transfer(owner, address(0), _tokenId);
        emit Supply(supplyBefore, supplyAfter);
    }
```

The main issue lies in this `approve`, which checks if the `msg.sender` is the owner and operator of the tokenId.

```solidity
 function approve(address _approved, uint256 _tokenId) public {
        address owner = idToOwner[_tokenId];
        // Throws if `_tokenId` is not a valid token
        require(owner != address(0), "owner not found");
        // Throws if `_approved` is the current owner
        require(_approved != owner, "Approved is already owner");
        // Check requirements
        bool senderIsOwner = (owner == msg.sender);
        bool senderIsApprovedForAll = (ownerToOperators[owner])[msg.sender];
        //@audit-issue Check will fail for the approved user who calls merge and withdraw
  ->>   require(senderIsOwner || senderIsApprovedForAll, "sender is not owner or approved");
        // Set the approval
        idToApprovals[_tokenId] = _approved;
        emit Approval(owner, _approved, _tokenId);
    }
```

The `approve` function implementation itself is correct if the external users call it but in this case the merge and withdraw is also using the same function which causes the issue.

### Note

The same issue was also submitted in the Velodrome c4 audit back in 2022. In that case, the problem was the same but the cause was different. <https://github.com/code-423n4/2022-05-velodrome-findings/issues/66>

### Recommendation

Instead of calling approve it is recommended to set the approval to address(0) or delete it directly.

```diff
 function _burn(uint256 _tokenId, uint256 _value) internal {
        address owner = ownerOf(_tokenId);

        // Update the total supply of deposited tokens
        uint256 supplyBefore = supply;
        uint256 supplyAfter = supplyBefore - _value;
        supply = supplyAfter;

        // Clear approval
-        approve(address(0), _tokenId);
+         idToApprovals[tokenId] = address(0);
        // Checkpoint for gov
        _moveTokenDelegates(delegates(owner), address(0), _tokenId);
        // Remove token
        _removeTokenFrom(owner, _tokenId);
        emit Transfer(owner, address(0), _tokenId);
        emit Supply(supplyBefore, supplyAfter);
    }
```

## Impact Details

approved user is unable to execute ordinary operations due to a logic flaw which can freeze the NFT for them temporarily.

As per this impact i belive the high is appropriate according to severity guidelines which accounts `Temporary freezing of NFT` as High.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L649>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L772>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L510>

## Proof of Concept

Paste this test inside `VotingEscrow.t.sol`. The test will pass on revert expectation as the merge is called by approved user.

```solidity
function testMergeTokensRevertEvenWhenCallerIsApproved() public {
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(admin, TOKEN_100K, MAXTIME / 2, false);
        // Approve both token to Beef
        hevm.startPrank(admin);
        veALCX.approve(beef, tokenId1);
        veALCX.approve(beef, tokenId2);
        hevm.stopPrank();
        hevm.startPrank(beef);

        uint256 lockEnd1 = veALCX.lockEnd(tokenId1);

        assertEq(lockEnd1, ((block.timestamp + MAXTIME) / ONE_WEEK) * ONE_WEEK);
        assertEq(veALCX.lockedAmount(tokenId1), TOKEN_1);

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
        voter.reset(tokenId2);

        uint256 unclaimedFluxBefore1 = flux.getUnclaimedFlux(tokenId1);
        uint256 unclaimedFluxBefore2 = flux.getUnclaimedFlux(tokenId2);
        hevm.expectRevert(abi.encodePacked("sender is not owner or approved"));
        veALCX.merge(tokenId1, tokenId2); // This will revert but it shouldn't
        hevm.stopPrank();
    }
```
