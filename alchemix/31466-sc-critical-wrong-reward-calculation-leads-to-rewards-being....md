# 31466 - \[SC - Critical] Wrong reward calculation leads to rewards being...

Submitted on May 20th 2024 at 00:08:35 UTC by @RandomSec for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31466

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

Hello,

upon running the tests supplied by you and changing them a bit I came across an issue that I couldn't find in the audit but that leads to most likely a high impact. The main reward calculation of `RevenueHandler` happens in `checkPoint()`. If a revenue token has a `poolAdapter` set other than `address(0)`, it is considered an alchemic-token. Then those tokens are `melted`, i.e., as far as I understand, exchanged and then the received value is added to `epochRevenues[currentEpoch][tokenConfig.debtToken]`. However, if it's not an alchemic-token, there is a miscalculation in the assigned rewards. If it is not, following code is executed (the else-clause):

```
// If the revenue token doesn't have a poolAdapter, it is not an alchemic-token
amountReceived = thisBalance;
// Update amount of non-alchemic-token revenue received for this epoch                    
epochRevenues[currentEpoch][token] += amountReceived;
```

However, all it does is applying the current balance to `epochRevenues[currentEpoch][token]`. The problem is, that this will constantly increase `epochRevenues[currentEpoch][token]` as users will not instantly claim all rewards. Additionally, it is wrong to assume that this comment "Update amount of non-alchemic-token revenue received for this epoch" is true because it could be that no rewards are received. This is easily proven by a test you provided already. All I did was to add

```
_jumpOneEpoch();
revenueHandler.checkpoint();
```

in the middle of it. You will see that the test will fail because of "Not enough revenue to claim" because it will now claim the "rewards" of two epochs, i.e., the `epochRevenues[currentEpoch][token]` increased by 2x the contract balance. The difference is clearly visible when considering that rewards added to alchemic-tokens are those funds received via an exchange of tokens (i.e., the amount can also be 0) vs. non-alchemic are simply increased by the token balance at time of `checkpoint()`-execution. This will break the whole reward calculation for non-alchemic tokens.

The fix would be to correctly track rewards for non-alchemic tokens by calculating the difference of actually received tokens vs. balance & claims. I haven't found any variables helpful for such a calculation so it may require rewriting the contract in order to fix this.

Please correct me if I am wrong.

Best RandomSec

(PS: I will add my wallet address later)

## Proof of Concept

```
// PoC 1: Copied & edited from testClaimNonAlchemicRevenue()
function testCustomTestClaimNonAlchemicRevenuePoC() external {
        uint256 revAmt = 1000e18;
        uint256 tokenId = _setupClaimableNonAlchemicRevenue(revAmt, bal);
        uint256 balBefore = IERC20(bal).balanceOf(address(this));

        assertEq(balBefore, 0, "should have no bal before claiming");
    
        // Added:
        _jumpOneEpoch();
        revenueHandler.checkpoint();

        uint256 claimable = revenueHandler.claimable(tokenId, bal);

        revenueHandler.claim(tokenId, bal, address(0), claimable, address(this));

        uint256 balAfter = IERC20(bal).balanceOf(address(this));

        assertEq(balAfter, claimable, "should be equal to amount claimed");
    }
```

PoC 2: More custom, proving claims of multiple accounts are not calculated correctly. In that PoC you see that it says the same amount as reward for all 3 epochs even though no epoch has given any rewards

```
function _initializeVeALCXPositionAs(uint256 lockAmt, address runAs) internal returns (uint256 tokenId) {
        veALCX.checkpoint();
        tokenId = _lockVeALCXAs(lockAmt, runAs);
    }

    function _lockVeALCXAs(uint256 amount, address runAs) internal returns (uint256 tokenId) {
        deal(address(bpt), runAs, amount);
        vm.startPrank(runAs);
        IERC20(bpt).approve(address(veALCX), amount);
        tokenId = veALCX.createLock(amount, MAXTIME, false);
        vm.stopPrank();
    }

    function testCustomClaimNonAlchemicRevenue() external {
        uint256 revAmt = 1000e18;
        uint256 tokenId = _setupClaimableNonAlchemicRevenue(revAmt, bal);

        address alice = address(0xa11ce);
        uint aliceTokenId = _initializeVeALCXPositionAs(10e18, alice);

        uint thisClaimable; uint aliceClaimable; uint epoch;
        uint thisLockedAmount; uint aliceLockedAmount; // Amounts locked by each account
        uint[3] memory epochRevenues;
        uint[3] memory epochs;

        // Testing PoC for 3 epochs
        for(uint i; i < 3; i++) {
            epoch = revenueHandler.currentEpoch();
            epochs[i] = epoch;
            epochRevenues[i] = revenueHandler.epochRevenues(epoch, bal);
            thisClaimable = revenueHandler.claimable(tokenId, bal);
            aliceClaimable = revenueHandler.claimable(aliceTokenId, bal);
            (thisLockedAmount, , , ) = veALCX.locked(tokenId);
            (aliceLockedAmount, , , ) = veALCX.locked(aliceTokenId);
            console.log("EP %s: Alice claimable: %s, vs. this: %s", epoch, aliceTokenId, thisClaimable);
            console.log("Locked this %s, vs alice %s", thisLockedAmount, aliceLockedAmount);

            console.log("Claimable this vs. balance revenueHandler:");
            console.log(revenueHandler.claimable(tokenId, bal));
            console.log(IERC20(bal).balanceOf(address(revenueHandler)));

            _jumpOneEpoch();
            revenueHandler.checkpoint();
        }

        console.log("Epochs: %s, %s & %s", epochs[0], epochs[1], epochs[2]);
        console.log("Epoch revenues: %s, %s & %s", epochRevenues[0], epochRevenues[1], epochRevenues[2]);
    }
```
