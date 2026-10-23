# #41640 \[SC-High] Stuck Rewards in StakeV2 Contract Due to Improper Handling of Leftover Tokens

**Submitted on Mar 17th 2025 at 07:59:42 UTC by @DoD4uFN for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41640
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The `StakeV2` contract interacts with the `Zapper` contract to facilitate reward claiming. However, the `zapOut`, `zapOutToToken0`, and `zapOutToToken1` functions in `Zapper` send leftover tokens to both the user and `_msgSender()`, which in this case is `StakeV2`. Since `StakeV2` has no mechanism to retrieve or use these tokens, they remain stuck in the contract indefinitely. This results in an unintended loss of user rewards and inefficient token management.

***

## Vulnerability Details

The issue arises from how the `Zapper` contract distributes leftover tokens after executing a reward claim. The `zapOutToToken0`, `zapOutToToken1`, and `zapOut` functions send a portion of the tokens to the receiver (the user) and another portion to `_msgSender()`, which is `StakeV2`. However, `StakeV2` does not have a function to recover or utilize these tokens, leading to a permanent loss of funds.

### Relevant Code Snippet

In `StakeV2`, the claim functions call `zapOut` functions from `Zapper`:

```solidity
uint256 receivedAmount = zapper.zapOutToToken0(msg.sender, swapData, unstakeParams, updatedRedeemParams);
```

Similarly, in `Zapper`, the `zapOutToToken0` function distributes leftover `token1Debt` to `_msgSender()`:

```solidity
_sendERC20Token(token1, _msgSender(), token1Debt);
```

Since `_msgSender()` is `StakeV2`, these tokens get stuck in the contract indefinitely.

***

## Impact Details

* **User funds are partially lost**: A portion of rewards intended for the user is permanently locked in `StakeV2`.
* **Contract accumulates unusable tokens**: Over time, the contract accumulates stuck tokens, which can affect the efficiency of reward distribution.

This issue could result in significant financial losses depending on the volume of rewards being claimed over time.

***

## References

[Zapper:zapOutToToken0](https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L261-L262)[Zapper:zapOutToToken1](https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L284-L285)[Zapper:zapOut](https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L352)

[StakeV2:claimRewardsInToken0](https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L356)[StakeV2:claimRewardsInToken1](https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L372)[StakeV2:claimRewardsInToken](https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L390-L391)

***

### Recommended Fix

Modify the `Zapper` contract to ensure that all leftover tokens are returned exclusively to the user, not `_msgSender()`. Example fix:

```solidity
_sendERC20Token(token1, receiver, token1Debt); // Ensure all tokens go to the receiver
```

## Proof of Concept

## Proof of Concept

1. Deploy `StakeV2` and `Zapper` contracts.
2. Stake tokens and accumulate rewards.
3. Call `claimRewardsInToken0()`, `claimRewardsInToken1()`, or `claimRewardsInToken()` with `swapData.inputAmount` less than `tokenXDebt`, which leads to `tokenXDebt` being non-zero.
4. Observe that part of the tokens sent to `_msgSender()` (i.e., `StakeV2`) are never recoverable.
5. Check the balance of `StakeV2`—tokens remain stuck indefinitely.

This PoC demonstrates that rewards are not fully transferred to users, leading to locked funds.
