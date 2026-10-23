# 31458 - \[SC - Critical] Invalid handling of epochs revenue for tokens t...

Submitted on May 19th 2024 at 20:57:08 UTC by @OxAlix2 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31458

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Smart contract unable to operate due to lack of token funds
* Contract fails to deliver promised returns, but doesn't lose value
* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

In the `RevenueHandler`, whenever a checkpoint is made the protocol takes the balance of all the revenue tokens, and handles them, each in a way according if that token has a corresponding pool adapter, here we have 2 options:

* The revenue token has a corresponding pool adapter: all of the current balance is "melted", i.e. transferred to the pool adapter, leaving the protocol with a 0 balance of that token.
* If not, the whole balance of the revenue token stays in the protocol, until users start coming in and claiming their rewards. After both of the above, the protocol saves the "current balance" in a mapping called `epochRevenues[currentEpoch][token]`, which states the revenue of that token in this epoch, reference <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol#L257-L264>.

This is okay for the first case, where a pool adapter exists for the token, however, it poses a serious issue for the second case. Because the tokens stay in the handler and aren't transferred (unlike the pool adapter), the remaining balance (remaining balance after some users claim their rewards) will still be calculated in the next epochs' revenue (which is saved in `epochRevenues[currentEpoch][token]`, which critically affects the accumulated rewards of users.

## Vulnerability Details

The epoch's revenue is used in `RevenueHandler::_claimable`, to calculate the percent of the rewards that should be claimed by the user, which will end up wrongly calculated as "deserved" rewards. Let's take an example, at epoch 1 the handler receives 100 DAI and is saved as epoch's revenue, the whole epoch passes by, no users have claimed rewards and no new funds come in, epoch 1 ends, `checkpoint` is called (it can be called by anyone permissionless), `thisBalance` will be 100 DAI and will be added as epoch 2 revenue (which is wrong as the handler didn't receive any new funds).

When users call `RevenueHandler::claim`, `_claimable` will be called and return a wrong value, because the 100 DAI (handler's balance) will be accumulated twice. The result will be a value, such that `claimable > handler's balance`.

So users won't be able to withdraw their "skyrocketed" rewards due to insufficient balance.

## Impact Details

Users will receive an inaccurate representation of their claimable rewards, and more critically won't be able to claim those rewards as they'll be more than what the handler has in hand.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol#L245>

## Proof of concept

```
function testInvalidEpochRevenue() public {
    uint256 revAmt = 1000e18;

    // Create a position
    uint256 tokenId = _initializeVeALCXPosition(10e18);

    // epoch passed
    _jumpOneEpoch();

    // revenue comes in and checkpoint
    _accrueRevenue(bal, revAmt);
    revenueHandler.checkpoint();

    // verify that epoch's revenue is 1000 DAI
    uint256 epoch1 = revenueHandler.currentEpoch();
    assertEq(revenueHandler.epochRevenues(epoch1, bal), revAmt);

    // another epoch passed
    _jumpOneEpoch();

    // checkpoint without accruing revenue
    revenueHandler.checkpoint();

    uint256 epoch2 = revenueHandler.currentEpoch();
    // verify that an epoch has passed
    assertGt(epoch2, epoch1);
    // new epoch's revenue is 1000 DAI, knowing that no new revenue was accrued
    assertEq(revenueHandler.epochRevenues(epoch2, bal), revAmt);

    uint256 claimable = revenueHandler.claimable(tokenId, bal);

    // user's claimable revenue is 2 * 1000 DAI
    // handler's DAI balance is 1000 DAI
    assertEq(claimable, revAmt * 2);
    assertEq(IERC20(bal).balanceOf(address(revenueHandler)), revAmt);

    // user can't claim their rewards as the handler doesn't have enough DAI
    vm.expectRevert(abi.encodePacked("Not enough revenue to claim"));
    revenueHandler.claim(tokenId, bal, address(0), claimable, address(this));
}
```
