# 31552 - \[SC - Insight] Lack of the validation for a Flash token protec...

Submitted on May 21st 2024 at 08:14:05 UTC by @Oxmuxyz for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31552

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

There is no the validation for a `Flash token protection` in the VotingEscrow#`getVotes()`, which potentially allow a malicious actor to launch a `Flash token` attack to manipulate the governance voting result.

## Vulnerability Details

Within the VotingEscrow contract, the `ownershipChange` storage would be defined to set the block number for a `tokenId` of veALCX when there is an ownership change like this:\
<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L82>

```solidity
    /// @notice Sets the block number for a tokenId when there is an ownership change
    mapping(uint256 => uint256) public ownershipChange;
```

Within the VotingEscrow#`_transferFrom()`, the current block number (`block.number`) would be stored into the `ownershipChange` storage. According to the [inline comment](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L942) of the VotingEscrow#`_transferFrom()`, this is done because of the **Flash token protection** like this:

> Set the block of ownership transfer (for Flash token protection)

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L942-L943>

```solidity
    function _transferFrom(address _from, address _to, uint256 _tokenId, address _sender) internal {
        ...
        // Set the block of ownership transfer (for Flash token protection)  ///<---------- @audit 
        ownershipChange[_tokenId] = block.number; ///<---------- @audit 
```

Within the VotingEscrow#`balanceOfToken()`, the block number of the `_tokenId` of veALCX, which is stored in the `ownershipChange` storage, would be checked. If the VotingEscrow#`balanceOfToken()` would be called in the **same block** with the veALCX token transfer (VotingEscrow#`_transferFrom()`), `0` would be returned. This reason is to prevent buying and voting in the same block and perform a **Flash token protection** (as the[ inline comment](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L942) explains in the VotingEscrow#`_transferFrom()` above). Then, the VotingEscrow#`_balanceOfTokenAt()` would internally be invoked like this:\
<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L367-L368>

```solidity
    /// @inheritdoc IVotingEscrow
    function balanceOfToken(uint256 _tokenId) external view returns (uint256) {
        if (ownershipChange[_tokenId] == block.number) return 0; ///<---------- @audit - it returns 0 if this function would be called in the same block with the veALCX token transfer. However, this check is not present in the VotingEscrow# getVotes() function.
        return _balanceOfTokenAt(_tokenId, block.timestamp); ///<---------- @audit 
    }
```

The VotingEscrow#`getVotes()` would be used to count the voting power of a given `account`. Within the VotingEscrow#`getVotes()`, the VotingEscrow#`_balanceOfTokenAt()` would internally be invoked like this: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L274>

```solidity
    /// @inheritdoc IVotingEscrow
    function getVotes(address account) external view override(IVotes, IVotingEscrow) returns (uint256) {
        uint32 nCheckpoints = numCheckpoints[account];
        if (nCheckpoints == 0) {
            return 0;
        }
        uint256[] memory _tokenIds = checkpoints[account][nCheckpoints - 1].tokenIds;
        uint256 votes = 0;
        uint256 tokenIdCount = _tokenIds.length;
        for (uint256 i = 0; i < tokenIdCount; i++) {
            uint256 tId = _tokenIds[i];
            votes = votes + _balanceOfTokenAt(tId, block.timestamp); ///<---------- @audit 
        }
        return votes;
    }
```

Within the VotingEscrow#`getVotes()` above, it is supposed to be checked whether or not the VotingEscrow#`getVotes()` is **not** called in the **same block** with the veALCX token transfer (VotingEscrow#`_transferFrom()`) **before** the VotingEscrow#`_balanceOfTokenAt()` is internally invoked ([like the VotingEscrow#`balanceOfToken()`](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L367)) for a Flash token protection.

However, within the VotingEscrow#`getVotes()` above, there is **no** validation to check whether or not the VotingEscrow#`getVotes()` is **not** called in the **same block** with the veALCX token transfer (VotingEscrow#`_transferFrom()`) **before** the VotingEscrow#`_balanceOfTokenAt()` is internally invoked.

This is problematic. Because, within the VotingEscrow#`getVotes()` above, the VotingEscrow#`_balanceOfTokenAt()` will internally be invoked with each tokenId (`tId`) of veALCX respectively in the for-loop. Since there is **no** validation **before** the VotingEscrow#`_balanceOfTokenAt()` is called, the voting power for these all tokenIds (`tld`s) of veALCX will be calculated as normal - even if the VotingEscrow#`getVotes()` would be called **in the same block** with the veALCX token transfer (VotingEscrow#`_transferFrom()`).

Hence, there will be **no** Flash token protection - if the voting mechanism integration is made with the VotingEscrow#`getVotes()`.

This lead to potentially allowing a malicious actor to launch a `Flash token` attack to manipulate the governance voting result.

**NOTE**: I acknowledge this is a view function and the impact is limited within the current codebase in the scope. However, I assume that the VotingEscrow#`getVotes()` will be used for actual voting and user voting powers will be determined based on that. I submitted it as medium severity even though voting manipulations would normally be high because of that assumption.

## Impact Details

This allow a malicious actor to launch a **Flash token** attack by utilizing the VotingEscrow#`getVotes()` - if the voting mechanism integration is made with the VotingEscrow#`getVotes()`.

## PoC

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L942-L943>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L367-L368>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L274>

## Recommendation

Within the VotingEscrow#`getVotes()`, consider implementing the validation to check whether or not the VotingEscrow#`getVotes()` is **not** called in the same block with the veALCX token transfer (VotingEscrow#`_transferFrom()`) before the VotingEscrow#`_balanceOfTokenAt()` is internally invoked like this:

```diff
    /// @inheritdoc IVotingEscrow
    function getVotes(address account) external view override(IVotes, IVotingEscrow) returns (uint256) {
        uint32 nCheckpoints = numCheckpoints[account];
        if (nCheckpoints == 0) {
            return 0;
        }
        uint256[] memory _tokenIds = checkpoints[account][nCheckpoints - 1].tokenIds;
        uint256 votes = 0;
        uint256 tokenIdCount = _tokenIds.length;
        for (uint256 i = 0; i < tokenIdCount; i++) {
            uint256 tId = _tokenIds[i];
-           votes = votes + _balanceOfTokenAt(tId, block.timestamp);
+           /// @audit - Only add the voting power (votes) if it is not the same block.
+           if (ownershipChange[_tokenId] != block.number) {
+               votes = votes + _balanceOfTokenAt(tId, block.timestamp);
+           }
        }
        return votes;
    }
```

## Proof of Concept
