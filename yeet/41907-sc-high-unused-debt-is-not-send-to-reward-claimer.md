# #41907 \[SC-High] Unused debt is not send to Reward Claimer

**Submitted on Mar 19th 2025 at 10:03:41 UTC by @Oxgritty for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41907
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

* When a user calls claimRewards(claimRewardsInNative/ claimRewardsInToken0/ claimRewardsInToken1/ claimRewardsInToken) function, his reward is converted into his choice of token, during this swapping the amount of token which was not used will be send to StakeV2.sol instead of the user.

## Vulnerability Details

* When a user calls one of these functions(claimRewardsInNative/ claimRewardsInToken0/ claimRewardsInToken1/ claimRewardsInToken) to claim rewards and inside these functions we call a relevant zapOut function(like `Zapper::zapOutNative` for `StakeV2::claimRewardsInNative` or `Zapper::zapOutToToken0` for `StakingV2::claimRewardsInToken0`) in Zapper.sol, Zapper.sol pays back the amount of unused debt to StakeV2.sol and StakeV2 does not send it back to the user.

## Impact Details

* Reward Claimer loses unused debt, as it is transferred to StakeV2.sol

## References

This issue is present in all 4 claimRewards function:

1. claimRewardsInNative\
   After [calling](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L340) `Zapper::zapInNative`, we should send unused debt back to the user, but we are not doing it.

* Same problem with these other 3 functions as well

2. claimRewardsInToken0
3. claimRewardsInToken1
4. claimRewardsInToken

## Proof of Concept

## Proof of Concept

### Step 0: Assumptions we are making

* Bob has rewards of 50e18 vaultShares that he wants to claim in Native Token(BERA) using `StakeV2::claimRewardsInNative`
* `zapOutNative` will turn 50e18 worth of vaultShares into the following Balances
  1. 45e18 (BERA)
  2. 2e18 (Token0)
  3. 3e18 (Token1)

### Step 1: Calling `StakeV2::claimRewardsInNative`

* Bob calls `StakeV2::claimRewardsInNative` with `amountToWithdraw=50e18` and function is successfully executed.

### Step 2: Checking Balances:

* For StakeV2
  1. Token0.balanceOf(address(StakeV2)) = 2e18
  2. Token1.balanceOf(address(StakeV2)) = 3e18
  3. BERA.balanceOf(address(StakeV2)) = 0
* For Bob
  1. Token0.balanceOf(address(Bob)) = 0
  2. Token1.balanceOf(address(Bob)) = 0
  3. BERA.balanceOf(address(Bob)) = 45e18
