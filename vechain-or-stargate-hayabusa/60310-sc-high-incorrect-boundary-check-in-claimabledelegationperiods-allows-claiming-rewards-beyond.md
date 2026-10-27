# 60310 sc high incorrect boundary check in claimabledelegationperiods allows claiming rewards beyond delegation end period

**Submitted on Nov 21st 2025 at 08:40:23 UTC by @Oxodus for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60310
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Protocol insolvency
  * Theft of unclaimed yield

## Description

## Brief/Intro

The `_claimableDelegationPeriods` function in the Stargate contract contains an incorrect boundary check (`endPeriod > nextClaimablePeriod`) that should use `>=` instead of `>`. This logic flaw allows delegators to claim rewards for periods beyond their delegation's end period when `endPeriod` equals `nextClaimablePeriod`, potentially claiming rewards they are not entitled to and draining rewards that should belong to other delegators or future periods.

## Vulnerability Details

In the `_claimableDelegationPeriods` function [lines 916-922](https://github.com/vechain/stargate-contracts/blob/jose/update-contracts-to-hayabusa/packages/contracts/contracts/Stargate.sol#L916) , the condition checking if a delegation has ended uses a strict greater than operator:

```solidity
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod  // @audit should be >=
) {
    return (nextClaimablePeriod, endPeriod);
}
```

The issue arises when `endPeriod` equals `nextClaimablePeriod`. In this case:

* The condition `endPeriod > nextClaimablePeriod` evaluates to `false`
* The function bypasses this branch and falls through to the next condition
* It returns `(nextClaimablePeriod, completedPeriods)` instead of `(nextClaimablePeriod, endPeriod)`
* This allows the delegator to claim rewards up to `completedPeriods`, which can be significantly greater than `endPeriod`

The correct logic should recognize that when `endPeriod >= nextClaimablePeriod`, there are still claimable periods from `nextClaimablePeriod` up to and including `endPeriod`. The current implementation incorrectly excludes the edge case where they are equal.

## Impact Details

### 1. Protocol Insolvency

The Stargate contract holds VTHO rewards to distribute to delegators based on their participation in completed periods. When exited delegators fraudulently claim rewards for periods they didn't participate in, they drain VTHO from the contract that should be reserved for legitimate claimants. If multiple delegators exploit this vulnerability across different validators and time periods, the cumulative theft can exceed the contract's VTHO balance. This leaves the protocol insolvent, resulting in a direct loss of funds for legitimate users.

### 2. Theft of Unclaimed Yield

Exited delegators can claim VTHO rewards for periods 7, 8, 9, etc., even though their delegation ended at period 6. These rewards rightfully belong to delegators who were actively staking during those periods. By exploiting the boundary check flaw, malicious actors directly steal yield that was earned by the effective stake of legitimate active delegators on the validator during post exit periods.

## References

<https://github.com/vechain/stargate-contracts/blob/jose/update-contracts-to-hayabusa/packages/contracts/contracts/Stargate.sol#L916>

## Proof of Concept

## Proof of Concept

Taking a scenario whereby:

* A delegator has `lastClaimedPeriod[tokenId] = 5`, making `nextClaimablePeriod = 6`
* The delegation was requested to exit and ended at `endPeriod = 6`
* The validator is still active with `completedPeriods = 9` (current period = 10) The protocol will behave as follows while checking for claimable rewards

1. Function enters the exit check:

   ```solidity
   endPeriod (6) != type(uint32).max ✓
   endPeriod (6) < currentValidatorPeriod (10) ✓
   endPeriod (6) > nextClaimablePeriod (6) ✗ FALSE
   ```
2. Condition fails, function continues to next branch:

   ```solidity
   nextClaimablePeriod (6) < currentValidatorPeriod (10) ✓
   ```
3. Function returns `(6, 9)` instead of `(6, 6)`
4. The delegator can now claim rewards for periods 6, 7, 8, and 9 via `_claimRewards`:

   ```solidity
   // Loop iterates from period 6 to 9 (inclusive)
   for (uint32 period = 6; period <= 9; period++) {
       claimableAmount += _claimableRewardsForPeriod($, _tokenId, period);
   }
   ```
5. After claiming, `lastClaimedPeriod[tokenId]` is set to 9, permanently encoding the incorrect state.
