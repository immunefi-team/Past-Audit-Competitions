# 31495 - \[SC - Critical] Users cannot claim rewards from RevenueHandler ...

Submitted on May 20th 2024 at 13:55:07 UTC by @OxAnmol for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31495

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

If a user extends their lock time to the maximum amount using `votingEscrow:updateLockTime` and sets `maxLockEnabled` to true, the `RevenueHandler:claim` function will fail when the user attempts to claim their rewards.

## Vulnerability Details

The `RevenueHandler:claim` function allows users to earn rewards every epoch for maintaining a veALCX position. Users can call this function at any time after locking BPT. The function calculates and distributes all the epoch rewards since the last claim to the user.

```solidity
   function claim(
        uint256 tokenId,
        address token,
        address alchemist,
        uint256 amount,
        address recipient
    ) external override {
        require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, tokenId), "Not approved or owner");

        uint256 amountBurned = 0;
->>     uint256 amountClaimable = _claimable(tokenId, token);
...SNIP...
  }

```

The `claim` function internally calls the `RevenueHandler:_claimable` function, which calculates the user's rewards for each epoch.

```solidity
function _claimable(uint256 tokenId, address token) internal view returns (uint256) {
        uint256 totalClaimable = 0;
        uint256 lastClaimEpochTimestamp = userCheckpoints[tokenId][token].lastClaimEpoch;
        if (lastClaimEpochTimestamp == 0) {
            uint256 lastUserEpoch = IVotingEscrow(veALCX).userFirstEpoch(tokenId);
            lastClaimEpochTimestamp = (IVotingEscrow(veALCX).pointHistoryTimestamp(lastUserEpoch) / WEEK) * WEEK - WEEK;
        }
        for (
            uint256 epochTimestamp = lastClaimEpochTimestamp + WEEK;
            epochTimestamp <= currentEpoch;
            epochTimestamp += WEEK
        ) {
            uint256 epochTotalVeSupply = IVotingEscrow(veALCX).totalSupplyAtT(epochTimestamp);
            if (epochTotalVeSupply == 0) continue;
            uint256 epochRevenue = epochRevenues[epochTimestamp][token];
L322::          uint256 epochUserVeBalance = IVotingEscrow(veALCX).balanceOfTokenAt(tokenId, epochTimestamp);
            totalClaimable += (epochRevenue * epochUserVeBalance) / epochTotalVeSupply;
        }
        return totalClaimable + userCheckpoints[tokenId][token].unclaimed;
    }

```

In line L322, the `VotingEscrow:balanceOfTokenAt` function checks for the user's underlying voting power/BPT. If the user has opted for the maximum lock, the balanceOfTokenAt function will return the maximum balance.

```solidity
  function _balanceOfTokenAt(uint256 _tokenId, uint256 _time) internal view returns (uint256) {
       ...SNIP...
            int256 biasCalculation = locked[_tokenId].maxLockEnabled
                ? int256(0)
                : lastPoint.slope * (int256(_time) - int256(lastPoint.ts));
           ...SNIP...
        }
    }

```

Let's say a user initially sets a lock end time of 90 days. After 2 epochs (4 weeks), they decide to extend the lock time to the maximum (1 year) by calling `updateLockTime` with `maxLockEnabled` set to true.

If they then try to claim the rewards for the first 2 epochs from the revenue handler, the `_claimable` function will behave as if they always had the maximum lock enabled. It will not account for the initial 90-day lock covering the first 2 epochs. The attempt to distribute rewards will fail because the contract will not have enough funds to distribute.

## Impact Details

Users will lose rewards. This is a loss of unclaimed yield for user which is considered as high impact.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L186>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L322>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L714>

## Proof of Concept

Paste this test in `RevenueHandler.t.sol`

We can see what are the steps taken here to reach the revert state.

```solidity
function testChangeLockTimeAfterTwoEpochAndClaimAll() public {
        //Lock BPT for 3 months
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, 3 * 30 days, false);
        uint256 revAmt = 1000e18;
        _accrueRevenueAndJumpOneEpoch(revAmt); // Deposit 1000 DAI

        hevm.startPrank(admin);
        veALCX.updateUnlockTime(tokenId, MAXTIME, true); // Enable max lock so we get max rewards from now onwards
        hevm.stopPrank();

        _accrueRevenueAndJumpOneEpoch(revAmt); // Deposit 1000 DAI

        hevm.startPrank(admin);
        expectError("Not enough revenue to claim");
        revenueHandler.claim(tokenId, alusd, address(alusdAlchemist), revenueHandler.claimable(tokenId, alusd), admin);

        hevm.stopPrank();
    }
```

## Output

```bash
Ran 1 test for src/test/RevenueHandler.t.sol:RevenueHandlerTest
[FAIL. Reason: revert: Not enough revenue to claim] testChangeLockTimeAfterTwoEpochAndClaimAll() (gas: 3287974)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 145.80s (113.26s CPU time)

Ran 1 test suite in 148.30s (145.80s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in src/test/RevenueHandler.t.sol:RevenueHandlerTest
[FAIL. Reason: revert: Not enough revenue to claim] testChangeLockTimeAfterTwoEpochAndClaimAll() (gas: 3287974)
```
