# 31480 - \[SC - High] Miscalculation of global bias

Submitted on May 20th 2024 at 05:12:32 UTC by @MahdiKarimi for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31480

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Permanent freezing of unclaimed yield
* Theft of unclaimed yield

## Description

## Brief/Intro

There is a problem in bias calculation which leads to differentiation between the sum of user balances and total supply (bias) in voting escrow

## Vulnerability Details

In case of maxLockEnabled, lock end is calculated by the following formula :

```
block.timestamp + MAXTIME) / WEEK) * WEEK
```

which rounds down unlock time to week, it means users with maxLockEnabled would have different lock duration and different veALCX balance for the same amount of lock amount, the problem arise when \_checkpoint function calculates old lock point ( bias and slope ) to update global bais

```
if (oldLocked.maxLockEnabled) oldLocked.end = ((block.timestamp + MAXTIME) / WEEK) * WEEK;
```

when user tries to merge or deposit in a lock and if max lock is enabled, \_checkpoint function updates old lock unlock time, and then calculates old lock point

```
oldPoint = _calculatePoint(oldLocked, block.timestamp);
```

then it uses oldPoint for updating global bias ( total veALCX supply )

```
lastPoint.bias += (newPoint.bias - oldPoint.bias);
```

when `_checkpoint` updates lock it calculates bias of old lock base on new lock duration which would be different ( up to 1 week ) from old lock original duration then it uses the difference between old lock bias and new lock bias to update global bias, since old bias is calculated again and due to round down explained above it may be different from original old lock bias, so global bias would be updated wrongly, it leads to a situation that sum of all user biases is not equal to global bias,

for example user creates a lock with 100 ALCX and receives 100 veALCX, now global bias is 100 ( considering he is only user ) , after sometime he adds 100 ALCX to the lock, and this time receives 220 veALCX for 200 ALCX ( 110 for old locked amount and 110 for new locked amount ), so old lock bias would be calculated 110, to update global bias delta of new and old lock would be added to global bias, in this case : 220 - 110 = 110, now global bias is 210 while user bias is 220 and has 220 veALCX.

## Impact Details

sum of all users balances wouldn't be equal to global bias, it affects some important protocol mechanisms like distribution of rewards at rewards distributor, revenue handler and voter, since they use global and user bias for distribution of rewards therefore some users would receive more rewads and some users can't receive rewards or loss of rewards in some cases

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1181> <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1177> <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1260>

## Proof of Concept

```
 function testSupplyandBalance() public {
        hevm.startPrank(admin);

        // assert user has no veALCX balance 
        assertEq(veALCX.balanceOf(admin), 0);

        // user creates a lock 
        uint256 tokenId = veALCX.createLock(TOKEN_1, THREE_WEEKS, true);

        // assert balance of user is equal to total supply since he's only user 
        uint256 balanceOfToken = veALCX.balanceOfToken(tokenId);
        uint256 totalSupply = veALCX.totalSupply(); 
        assertEq(balanceOfToken, totalSupply);

        // jump to 6 days later 
        // add deposit to lock  
        hevm.warp(block.timestamp +  6 days);
        veALCX.depositFor(tokenId, TOKEN_1);

        // retrive new user balance and total supply 
        // as we see balance of user is more or less than total supply ( despite that he is only user )
        uint256 balanceOfTokenEnd = veALCX.balanceOfToken(tokenId);
        uint256 totalSupplyEnd = veALCX.totalSupply(); 
        assert(balanceOfTokenEnd < totalSupplyEnd || balanceOfTokenEnd > totalSupplyEnd);
    }
```
