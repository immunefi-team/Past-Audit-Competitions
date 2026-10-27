# 51613 sc medium yield tokens can be stuck in arctokenpurchase plumestakingrewardtreasury or other defi protocols when distributeyield is called&#x20;

**Submitted on Aug 4th 2025 at 11:42:59 UTC by @WinSec for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51613
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief / Intro

As the title suggests, yield tokens can be stuck in `ArcTokenPurchase`, `PlumeStakingRewardTreasury` or other DeFi protocols when `distributeYield` is called. This can lead to these yield tokens being permanently lost.

### Vulnerability Details

Whenever `distributeYield` is called, it distributes the yield tokens to all the eligible holders (users who haven't been blacklisted) of the Arc Token.

Because yield is distributed to all holders, tokens may be sent to smart contracts that cannot withdraw arbitrary ERC-20 tokens. Examples include sale/purchase contracts, staking treasuries that only support specific reward tokens, DEX pair contracts, lending pools, etc. If those contracts do not implement functionality to forward or withdraw the received yield token, the tokens will be stuck.

{% stepper %}
{% step %}

### ArcTokenPurchase contract

When ArcTokens are sent to the `ArcTokenPurchase` contract (e.g., tokens left there for sale), `distributeYield` can send yield tokens into that contract. `ArcTokenPurchase` has no functionality to remove arbitrary tokens other than the purchase token or the arc token, so yield tokens different from those will be stuck.
{% endstep %}

{% step %}

### PlumeStakingRewardTreasury

If ArcTokens are used as a reward token and held in `PlumeStakingRewardTreasury`, but the yield token distributed is not one of the reward tokens managed by the treasury, then the yield tokens will accumulate with no method to withdraw them. `distributeReward` only distributes configured reward tokens to stakers, so unrelated yield tokens can become permanently stuck.
{% endstep %}

{% step %}

### Third-party integrations (DEXs, lending platforms, etc.)

If ArcTokens are used in third-party contracts (e.g., DEX pair contracts, lending pools), yield tokens distributed to those contracts may not be withdrawable by anyone, leaving them permanently stuck.
{% endstep %}
{% endstepper %}

## Impact Details

Yield tokens will be sent to contracts where they cannot be withdrawn unless those contracts explicitly support withdrawal of arbitrary ERC-20 tokens. This can lead to permanent loss of yield tokens (theft of unclaimed yield). Severity is classified as Medium.

## References

<https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/arc/src/ArcToken.sol#L388>

## Proof of Concept

{% stepper %}
{% step %}

### ArcTokenPurchase example

1. Arc tokens are sent to the `ArcTokenPurchase` contract to enable purchase via its `buy` function.
2. `distributeYield` is called. Yield tokens are distributed to all holders, including `ArcTokenPurchase`.
3. `ArcTokenPurchase` lacks a mechanism to withdraw arbitrary ERC-20 tokens, so the yield tokens remain stuck in the contract.
   {% endstep %}

{% step %}

### PlumeStakingRewardTreasury example

1. Arc tokens are used as a reward token and held by `PlumeStakingRewardTreasury`.
2. If the distributed yield token is not configured as a reward token in the treasury, `distributeYield` sends those tokens to the treasury.
3. There is no function to withdraw non-reward yield tokens, so they remain permanently stuck.
   {% endstep %}

{% step %}

### DEX / third-party contract example

1. Arc tokens are provided to a DEX pair or other third-party contract.
2. `distributeYield` sends yield tokens to all ArcToken holders, including that contract.
3. The third-party contract typically lacks logic to forward or withdraw arbitrary tokens it receives, so the yield tokens become permanently stuck.
   {% endstep %}
   {% endstepper %}
