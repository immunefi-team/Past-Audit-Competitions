# #41639 \[SC-Insight] Cross-Vault Reward Arbitrage in StakeV2 Allows Yield Theft

**Submitted on Mar 17th 2025 at 07:55:55 UTC by @DoD4uFN for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41639
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The `StakeV2` contract allows users to claim staking rewards in the form of shares from different `MoneyBrinter` vaults. However, due to improper tracking of reward distributions, users can claim rewards from any whitelisted vault, regardless of which vault was used for their reward distribution. This enables users to exchange lower-value shares for higher-value shares, effectively stealing yield from other stakers.

## Vulnerability Details

In `StakeV2`, when the manager calls `executeRewardDistributionYeet`, the Zapper contract performs a series of operations to convert `YEET` tokens into vault shares, which are then allocated as rewards to stakers. However, the contract does not properly associate users’ earned rewards with the specific `MoneyBrinter` vault from which they originated.

* The `_handleVaultShares` function updates the total vault shares but does not track which vault the shares belong to.
* The `_verifyAndPrepareClaim` function allows users to withdraw their earned shares from any whitelisted `MoneyBrinter` vault, without verifying that their rewards originated from that specific vault.
* As a result, a user who earned shares from a vault with a lower share price can claim shares from a vault with a higher share price, effectively gaining more value than they are entitled to.

### Example Scenario:

1. A user stakes `YEET` and earns 10 shares in Vault A, where each share is worth $5 (total $50).
2. The user calls `claimRewardsInNative`, but instead of withdrawing from Vault A, they specify Vault B, where each share is worth $10.
3. The contract does not verify the vault mismatch and allows the user to withdraw 10 shares from Vault B, giving them $100 instead of $50.
4. This results in a net loss of $50 to the protocol and other stakers.

The issue stems from the lack of vault-specific reward tracking in `_handleVaultShares` and `_verifyAndPrepareClaim`, allowing a user to arbitrage reward shares across different vaults.

## Impact Details

This vulnerability allows users to unfairly claim higher-value rewards at the expense of other stakers, leading to direct yield loss to the honest stakers. Depending on the total TVL of the protocol, the impact could lead to significant fund misallocations.

## References

[StakeV2.sol:executeRewardDistributionYeet](https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L153-L180)[StakeV2.sol:\_handleVaultShares](https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L206-L214)[StakeV2.sol:claimRewardsInNative](https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L327-L343)[StakeV2.sol:\_verifyAndPrepareClaim](https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L419-L435)

### Recommended Fix

* Track rewards on a per-vault basis to ensure users can only claim from the vault where their rewards originated.
* Modify `_handleVaultShares` to store vault-specific reward information.
* Update `_verifyAndPrepareClaim` to enforce reward claims from the correct vault.

## Proof of Concept

## Proof of Concept

1. Manager distributes rewards into a high-value `MoneyBrinter` vault.
2. User stakes tokens.
3. Manager distributes rewards into a low-value `MoneyBrinter` vault.
4. User earns shares of the low-value vault.
5. User claims rewards but specifies a high-value `MoneyBrinter` vault.
6. Contract fails to validate vault-specific rewards and allows the user to claim overvalued shares.
7. User profits by swapping low-value shares for high-value ones, stealing yield from other stakers.
