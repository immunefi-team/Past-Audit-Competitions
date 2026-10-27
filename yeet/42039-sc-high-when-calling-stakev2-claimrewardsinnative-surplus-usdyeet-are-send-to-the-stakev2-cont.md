# #42039 \[SC-High] When calling \`StakeV2::claimRewardsInNative()\` surplus $YEET are send to the StakeV2 contract instead of the user

**Submitted on Mar 20th 2025 at 08:26:24 UTC by @BenR for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42039
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

When a user calls `StakeV2::claimRewardsInNative` to claim his rewards in $BERA tokens, surplus $YEET tokens are send to the StakeV2 contract instead of the user. This results in a loss of the surplus tokens for the user.

## Vulnerability Details

When staking $YEET in the StakeV2 contract, users are earning rewards in form of vault shares from the Trifecta vault. If a user wants to claim his rewards in $BERA tokens he calls `StakeV2::claimRewardsInNative()`. The function checks if the user has earned the provided amount of shares and calls `Zapper:: zapOutNative()`. This function

* withdraws the provided amount of share amount from the Trifecta vault
* unstakes the received LP tokens from the $YEET/$WBERA KodiakVault and receives the corresponding $YEET and $WBERA tokens
* swaps the $YEET tokens to $WBERA tokens based on swap parameters provided by the user
* unwraps the $WBERA to $BERA and sends it to the user
* transfers any surplus $YEET tokens not consumed by the swap using the `_clearUserDebt` function

```solidity
_clearUserDebt(token0, token1, token0Debt, token1Debt, _msgSender());
```

The issue arises from the fact that the receiver of the surplus $YEET tokens is set to `_msgSender()` which is the `StakeV2` contract and not the user.

## Impact Details

Because the surplus $YEET tokens are send to `_msgSender()` which is the `StakeV2` contract, their value is lost for the user.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L622>

## Proof of Concept

## POC (step by step)

* user want to claim his rewards in BERA and calls `StakeV2::claimRewardsInNative` with the amount of 100 shares
* the 100 shares are withdrawn from the TrifectaVault returning 100 LP tokens of the `$YEET/$WBERA KodiakVault`
* the 100 LP tokens are unstaked from the `$YEET/$WBERA KodiakVault` and the Zapper contract receives 500$WBERA and 500,000 $YEET
* based on the parameters provided by the user 400,000 $YEET are swapped to 400 $WBERA
* all 900 $WBERA are unwrapped to $BERA and send to the user
* the surplus 100.000 $YEET are send to the StakingV2 contract instead of the user resulting in a loss of 100.000 $YEET for the user
