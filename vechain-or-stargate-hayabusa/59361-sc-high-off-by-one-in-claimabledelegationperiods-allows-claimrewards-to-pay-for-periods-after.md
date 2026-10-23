# 59361 sc high off by one in claimabledelegationperiods allows claimrewards to pay for periods after delegation end over claim theft of unclaimed yield

**Submitted on Nov 11th 2025 at 14:25:06 UTC by @daxun for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59361
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief / Intro

`Stargate.sol` computes the range of claimable periods for a delegated token and then pays VTHO rewards for every claimable period. Due to an off-by-one conditional in `_claimableDelegationPeriods`, when the delegation `endPeriod` equals the token's `nextClaimablePeriod`, the function fails to cap the last claimable period at `endPeriod`. If the validator’s `completedPeriods` later advances past `endPeriod`, `claimRewards()` will allow claims up to `completedPeriods` — including periods after the delegation ended — causing the contract to pay rewards the token wasn’t entitled to.

### Vulnerability Details

Location: `packages/contracts/contracts/Stargate.sol` — function `_claimableDelegationPeriods(...)`.\
<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L879-L934>

Problematic code fragment:

```solidity
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod
) {
    return (nextClaimablePeriod, endPeriod);
}
```

* `endPeriod` is the period when the delegation ends (set to `type(uint32).max` when no exit requested).
* `nextClaimablePeriod` is `lastClaimedPeriod + 1`, adjusted to delegation `startPeriod` if needed.
* `currentValidatorPeriod = completedPeriods + 1` (the ongoing period).

When `endPeriod == nextClaimablePeriod` (the equal case), the `endPeriod > nextClaimablePeriod` check fails. Execution falls through and later the function may return `(nextClaimablePeriod, completedPeriods)` (the validator's `completedPeriods`) if `nextClaimablePeriod < currentValidatorPeriod`. If `completedPeriods > endPeriod`, the claimable window now includes periods after `endPeriod`. The reward calculation `_claimableRewardsForPeriod()` uses stored `effectiveStake` values and does not verify per-period participation; therefore, the token owner receives VTHO for those extra periods.

This is a pure logic bug (off-by-one) — no race conditions, no privileged access, and reproducible with protocol state manipulation.

## Impact Details

* **Category:** Theft of unclaimed yield (in-scope, High severity).
* **Concrete impact:** Token owners can receive VTHO for periods during which their token was not delegated. The attacker can:
  * Create a delegation with `startPeriod = X`, `endPeriod = X` (equal to `nextClaimablePeriod`), or otherwise arrange state so `endPeriod == nextClaimablePeriod`.
  * Let `completedPeriods` advance beyond `endPeriod` (natural protocol progression).
  * Call `claimRewards()` and receive rewards through `completedPeriods`, including periods after `endPeriod`.
* **Financial impact:** Any amount of VTHO paid per period × number of over-claimable periods. Depending on production reward sizes and multiple tokens this can be materially high. This is direct protocol loss (funds transfer out of protocol-controlled VTHO).
* **Exploit prerequisites:** None privileged. Requires only an attacker-controlled token and normal protocol progression of `completedPeriods`. Fully reproducible via local test harness or mainnet fork.

## References

* Contract: `packages/contracts/contracts/Stargate.sol` (function `_claimableDelegationPeriods`)
* Relevant functions: `_claimRewards`, `_claimableRewards`, `_claimableRewardsForPeriod` in same file.
* Program scope: `Stargate.sol` is explicitly in-scope per the audit competition.

## Proof of Concept

{% stepper %}
{% step %}

### Setup

1. Deploy contracts (or use a mock):
   * `Stargate` (initialize with `protocolStakerContract` pointing to a mock `ProtocolStaker` and `stargateNFTContract` pointing to a mock `StargateNFT`).
   * `MockProtocolStaker` should allow controlling `completedPeriods` and provide delegation period responses.
   * `MockStargateNFT` should allow minting a token whose `vetAmountStaked` and `level` are known.
     {% endstep %}

{% step %}

### Create delegation

2. Create a delegation for token T:
   * Ensure at delegation time `completedPeriods = N`.
   * On delegation, Stargate sets `lastClaimedPeriod = completedPeriods + 1` (which equals `N+1`), so `nextClaimablePeriod = N+2` normally — but craft mocks so that the delegation start/end results in `nextClaimablePeriod == endPeriod`. Achieve this by configuring the mock `ProtocolStaker` to set delegation `startPeriod` and `endPeriod` appropriately during PoC.
     {% endstep %}

{% step %}

### Signal exit

3. Signal an exit on the delegation such that `endPeriod == nextClaimablePeriod` (the equal case). For example, set `endPeriod` to the delegation’s start period.
   {% endstep %}

{% step %}

### Advance validator periods

4. Advance `completedPeriods` on the mock to a value greater than `endPeriod`. Example: set `completedPeriods = endPeriod + 2`.
   {% endstep %}

{% step %}

### Claim rewards

5. Call `Stargate.claimRewards(tokenId)`.
   {% endstep %}

{% step %}

### Observe over-claim

6. Observe that `claimRewards()` computes `lastClaimablePeriod = completedPeriods` and transfers VTHO rewards for periods including those > `endPeriod`. On a mock, assert that transfer amount corresponds to number of over-claimed periods.
   {% endstep %}
   {% endstepper %}

## Minimal deterministic test skeleton (conceptual)

* Use a `MockProtocolStaker` that:
  * Returns controlled `completedPeriods` via `getValidationPeriodDetails`.
  * When `addDelegation` is called, returns a delegation id with `startPeriod = completedPeriods + 1` and `end = type(uint32).max`.
  * When `signalDelegationExit(delegationId)` is called, sets `end = start` (so `end == nextClaimablePeriod`).
  * `getDelegatorsRewards(validator, period)` returns fixed reward per period (e.g., `1e18`) so the PoC can assert transfer amounts.
* Use a `MockStargateNFT` that:
  * `mint()` returns a token with `vetAmountStaked` and `level` that produce a stable `effectiveStake`.
  * `ownerOf` returns the attacker address.
* Run the sequence: `stakeAndDelegate()` -> `signalDelegationExit()` -> advance `completedPeriods` -> `claimRewards()` -> assert VTHO transfer includes periods after `endPeriod`.

## Notes

* The root cause is an off-by-one comparison: the `>` check on `endPeriod` excludes the equal case, which should be capped at `endPeriod` to avoid paying out periods after delegation end.
* Fix should ensure `endPeriod >= nextClaimablePeriod` (or equivalently adjust comparisons) so the returned last claimable period is at most `endPeriod` when `endPeriod` is not `type(uint32).max`.
