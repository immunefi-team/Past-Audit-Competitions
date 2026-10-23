# 31521 - \[SC - Medium] Early return in RewardsDistributorclaim can cau...

Submitted on May 21st 2024 at 00:52:07 UTC by @Django for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31521

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

A user can lose ETH that they've sent to `RewardsDistributor` to compound their ALCX rewards. This can occur if a user accidently attempts to claim their reward multiple times. While the impact is severe, its likelihood is low, leading to medium severity.

## Vulnerability Details

A user is able to compound their rewards by sending ETH directly with the `claim()` call which will add liquidity to the WETH/ALCX pool. However, if their transaction gets stuck and they attempt to send another transaction, the RewardsDistributor contract will swallow the ETH and neither call will revert.

This is due to the fact that the function returns early if the user has no ALCX rewards.

```
    function claim(uint256 _tokenId, bool _compound) external payable nonReentrant returns (uint256) {
        if (!_compound) {
            require(msg.value == 0, "Value must be 0 if not compounding");
        }

...


        uint256 alcxAmount = _claim(_tokenId, votingEscrow, _lastTokenTime);


        // Return 0 without reverting if there are no rewards
        if (alcxAmount == 0) return alcxAmount;

...


        if (_compound) {
            (uint256 wethAmount, uint256[] memory normalizedWeights) = amountToCompound(alcxAmount);


            require(
                msg.value >= wethAmount || WETH.balanceOf(msg.sender) >= wethAmount,
                "insufficient balance to compound"
            );
```

As seen above, if the user wants to compound their rewards, they send the proper amount of ETH (or in excess) which bundles the ETH and ALCX and stakes the resulting BPT in the VotingEscrow contract on behalf of the user.

If the user sends two transactions with positive ETH values attached, the early return statement `if (alcxAmount == 0) return alcxAmount;` will trigger and the user's ETH will not be refunded, nor will it even be checked for proper amount.

## Impact Details

* Users' ETH can be stuck in the RewardDistributor

## Recommendation

Add this clause to check `msg.value`.

```
 if (alcxAmount == 0) {
    require(msg.value == 0);
    return alcxAmount;
}
```

## Output from POC

```
[PASS] testLostETHInClaim() (gas: 4022440)
Logs:
  ETH balance before compound: 100000000000000000000
  ETH balance after compound: 89715002151372114702
  ETH balance after second compound: 69715002151372114702
  ETH balance after third compound: 49715002151372114702
  Contract has swallowed 40 ETH
```

## Proof of Concept

```
function testLostETHInClaim() public {
        hevm.deal(admin, 100 ether);
        uint256 period = minter.activePeriod();

        // Create a veALCX token and vote to trigger voter rewards
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

        // Move forward a week relative to period
        hevm.warp(period + nextEpoch);
        voter.distribute();

        hevm.startPrank(admin);

        console.log("ETH balance before compound: %i", address(admin).balance);
        distributor.claim{ value: 20 ether }(tokenId, true);
        console.log("ETH balance after compound: %i", address(admin).balance);

        distributor.claim{ value: 20 ether }(tokenId, true);
        console.log("ETH balance after second compound: %i", address(admin).balance);

        distributor.claim{ value: 20 ether }(tokenId, true);
        console.log("ETH balance after third compound: %i", address(admin).balance);

        console.log("Contract has swallowed 40 ETH");

        hevm.stopPrank();
    }
```
