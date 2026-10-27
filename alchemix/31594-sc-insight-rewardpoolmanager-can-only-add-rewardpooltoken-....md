# 31594 - \[SC - Insight] RewardPoolManager can only add RewardPoolToken ...

Submitted on May 21st 2024 at 15:49:14 UTC by @hulkvision for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31594

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardPoolManager.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

I have mentioned two issues that would be nice to fix in this report.

1. RewardPoolManager can only add RewardPoolToken and has a limit of maximum limit of 10 reward tokens and does not have way to remove it.
2. In `VotingEscrow.sol` `setClaimFee` does not have any upper limit check on fee amount set by the admin.

## Vulnerability Details #1

RewardPoolManager can only add RewardPoolToken and has a limit of maximum limit of 10 reward tokens and does not have way to remove it, so if a reward token becomes insolvent or its value decreased to zero, the contract will have no way to remove these tokens.

In `RewardPoolManager.sol`

```
function _addRewardPoolToken(address token) internal { //@audit-ok
        if (!isRewardPoolToken[token] && token != address(0)) {
            require(rewardPoolTokens.length < MAX_REWARD_POOL_TOKENS, "too many reward pool tokens");

            isRewardPoolToken[token] = true;
            rewardPoolTokens.push(token);
        }
    }
```

## Impact Details #1

* Contract will not be to remove tokens that have become malicious or their value has decreased to zero.

## Vulnerability Detail #2

In `VotingEscrow.sol`

```
function setClaimFee(uint256 _claimFeeBps) external { //@audit no upper limit check present
        require(msg.sender == admin, "not admin");
        claimFeeBps = _claimFeeBps;
        emit ClaimFeeUpdated(_claimFeeBps);
    }
```

`setClaimFee` function is used by admin to set the claim fee for claiming ALCX rewards early, it is used by `RewardsDistributor.sol` contract to claim fees. This function does not have a upper limit check present on how much fee can be claimed.

## Impact Details #2

* Allowing arbitrary fee without upper limit could reduce user trust in the protocol.

## Proof of Concept

Add this test to `src/test/VotingEscrow.t.sol` and run test with `forge test --mt testPocArbitraryClaimFees --rpc-url $RPC_URL`

```
function testPocArbitraryClaimFees() public {
    veALCX.setClaimFee(1000000000);
        assertEq(veALCX.claimFeeBps(), 1000000000, "incorrect claim fee");
}
```
