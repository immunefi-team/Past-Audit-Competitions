# 30694 - \[SC - Low] Users approved for a single token id cannot wit...

Submitted on May 5th 2024 at 00:02:52 UTC by @imsrybr0 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30694

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Users approved for a single token id cannot `withdraw` or `merge` for that token id.

## Vulnerability Details

```solidity
    function _isApprovedOrOwner(address _spender, uint256 _tokenId) internal view returns (bool) {
        address owner = idToOwner[_tokenId];
        bool spenderIsOwner = owner == _spender;
        bool spenderIsApproved = _spender == idToApprovals[_tokenId];
        bool spenderIsApprovedForAll = (ownerToOperators[owner])[_spender];
        return spenderIsOwner || spenderIsApproved || spenderIsApprovedForAll;
    }

    function merge(uint256 _from, uint256 _to) external {
        //...
        require(_isApprovedOrOwner(msg.sender, _from), "not approved or owner");
        require(_isApprovedOrOwner(msg.sender, _to), "not approved or owner");
        // ...

        _burn(_from, value0);
        _depositFor(_to, value0, end, _locked1.maxLockEnabled, _locked1, DepositType.MERGE_TYPE);
    }

    function withdraw(uint256 _tokenId) public nonreentrant {
        require(_isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");

       // ...

        _burn(_tokenId, value);

        emit Withdraw(msg.sender, _tokenId, value, block.timestamp);
    }

    function _burn(uint256 _tokenId, uint256 _value) internal {
        // ...
        approve(address(0), _tokenId);
        // ...
    }

    function approve(address _approved, uint256 _tokenId) public {
        address owner = idToOwner[_tokenId];
        // Throws if `_tokenId` is not a valid token
        require(owner != address(0), "owner not found");
        // Throws if `_approved` is the current owner
        require(_approved != owner, "Approved is already owner");
        // Check requirements
        bool senderIsOwner = (owner == msg.sender);
        bool senderIsApprovedForAll = (ownerToOperators[owner])[msg.sender];
        require(senderIsOwner || senderIsApprovedForAll, "sender is not owner or approved");
        // Set the approval
        idToApprovals[_tokenId] = _approved;
        emit Approval(owner, _approved, _tokenId);
    }
```

Both `withdraw` and `merge` check if the `msg.sender` is owner of the given token id or is approved to use it (either for all tokens from the same owner of specifically the one being used).

They will also `burn` the token after carrying on their logic and clear its approvals.

Approvals are cleared using `approve(address(0), _tokenId)` which will fail if a `msg.sender` is only approved for that token id specifically.

## Impact Details

* User approved for a single token cannot `withdraw` or `merge`.
* Users need to give permission for all their tokens to another user when they want that user to carry `withdraw` or `merge` operations for them.

## Recommendation

Use the `_clearApproval(owner, _tokenId)` to clear the approvals in the `_burn` function.

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L741-L775>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L618-L651>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L826-L832>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L501-L514>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L1558-L1574>

## Proof of Concept

```solidity
    function testApproveSingleAndMerge() public {
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(beef, TOKEN_1, MAXTIME, false);

        hevm.prank(admin);
        veALCX.approve(beef, tokenId1);

        hevm.prank(beef);
        veALCX.merge(tokenId1, tokenId2);

        assertEq(veALCX.lockedAmount(tokenId2), TOKEN_1 + TOKEN_1);
    }
```
