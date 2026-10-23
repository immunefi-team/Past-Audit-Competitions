# 59358 sc high off by one error in reward claim logic allows delegators to steal vtho for periods after delegation ended

**Submitted on Nov 11th 2025 at 13:33:27 UTC by @Tomioka for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59358
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

The VeChain Stargate protocol contains an off-by-one boundary check error in the reward claiming logic that allows users to claim VTHO rewards for periods after their delegation has ended. When a user has claimed rewards up to their final delegation period and then the delegation ends, a faulty inequality check (`>` instead of `>=`) causes the clamping logic to fail, allowing the user to claim rewards for all subsequent periods up to the validator's current completed period. This enables direct theft of VTHO rewards that should be distributed to active delegators.

## Vulnerability Details

When users claim rewards, the protocol calculates which periods are claimable using the `_claimableDelegationPeriods` function. This function should ensure users can only claim rewards for periods during which they were actively delegated. For ended delegations, it should clamp the maximum claimable period to the delegation's `endPeriod`.

The vulnerable snippet:

```solidity
function _claimableDelegationPeriods(
    StargateStorage storage $,
    uint256 _tokenId,
    uint256 _delegationId
) internal view returns (uint32 firstClaimablePeriod, uint32 lastClaimablePeriod) {
    // ... [code to calculate nextClaimablePeriod] ...

    // Attempt to clamp to endPeriod
    if (
        endPeriod != type(uint32).max &&
        endPeriod < currentValidatorPeriod &&
        endPeriod > nextClaimablePeriod  // BUG: Should be >=
    ) {
        return (nextClaimablePeriod, endPeriod);
    }

    // Falls through when endPeriod == nextClaimablePeriod
    if (nextClaimablePeriod < currentValidatorPeriod) {
        return (nextClaimablePeriod, completedPeriods); // Returns beyond endPeriod!
    }
    // ...
}
```

Location: [Stargate.sol:917-921](broken://pages/ed14e4025cdeb5e726ede3c752eb32f2f3462486#L917-L921)

{% stepper %}
{% step %}

### The Bug Explained

When `endPeriod == nextClaimablePeriod` the condition `endPeriod > nextClaimablePeriod` is false, so the clamping branch is bypassed and the function falls through to the general case which returns `completedPeriods` (the validator's current period). This allows the caller to claim far beyond their delegation end.

This equality case commonly occurs when:

* A user has claimed all rewards up to period N-1
* Their delegation ends at period N (endPeriod = N)
* The next claimable period is N (nextClaimablePeriod = N)
  {% endstep %}
  {% endstepper %}

Why this is exploitable:

* Users who claim regularly are more vulnerable because their `nextClaimablePeriod` often equals `endPeriod`.
* After delegation end, a malicious final claim can request rewards up to the validator's current period, stealing VTHO intended for active delegators.

## Impact Details

* Direct theft of VTHO from the protocol's reward distribution pool.
* Stolen amount = (user\_stake / validator\_total\_stake) × reward\_per\_period × overclaimed\_periods.
* Theft compounds linearly with time elapsed after delegation end.

Realistic example (from report):

* User stake: 500,000
* Validator total stake: 10,000,000
* Reward per period: 100,000 VTHO
* User waits 10 periods after delegation end

Legitimate reward (period 10 only): (500,000 / 10,000,000) × 100,000 = 5,000 VTHO

Stolen reward (periods 10–19): (500,000 / 10,000,000) × 100,000 × 10 = 50,000 VTHO

Total overclaim: 45,000 VTHO stolen (9× legitimate amount)

Affected parties:

* Protocol: loses VTHO from reward pool
* Active delegators: receive reduced rewards
* Honest users who claim promptly: paradoxically more vulnerable

Scale:

* Any user can exploit this after exiting delegation
* Incentivizes delaying final claim to maximize overclaim

## References

* Vulnerable function: \[Stargate.sol:\_claimableDelegationPeriods]`packages/contracts/contracts/Stargate.sol#L917-L921`
* Claim rewards function: \[Stargate.sol:claimRewards]`packages/contracts/contracts/Stargate.sol`

## Proof of Concept

### Prerequisites

* Normal user operations only:
  * User has an active delegation
  * User claims rewards regularly
  * User requests exit or delegation naturally ends
  * Validator continues operating (periods advance)

### Running the Proof of Concept

From the `packages/contracts` directory run the Hardhat test with the environment variables used during testing.

Commands:

```bash
# change to the contracts package
cd packages/contracts

# recommended: ensure Docker Desktop is running

# then run the PoC using Hardhat and the vechain_solo network
VITE_APP_ENV=local \
MNEMONIC="denial kitchen pet squirrel other broom bar gas better priority spoil cross" \
npx hardhat test --network vechain_solo test/integration/POC-ClaimablePeriodOverclaim.test.ts --show-stack-traces
```

Or set environment variables first:

```bash
cd packages/contracts
export VITE_APP_ENV=local
export MNEMONIC="denial kitchen pet squirrel other broom bar gas better priority spoil cross"
npx hardhat test --network vechain_solo test/integration/POC-ClaimablePeriodOverclaim.test.ts --show-stack-traces
```

### Full PoC (test file)

```typescript
// (Full PoC test source as provided in the original report — omitted here for brevity)
// See original report for the complete TypeScript test code.
```

(The report contains the full TypeScript PoC test that demonstrates lastClaimable > delegation end after exit.)

## Recommended Fix

Primary fix — change the strict `>` to `>=`:

```solidity
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod >= nextClaimablePeriod  // Fixed: Changed to >=
) {
    return (nextClaimablePeriod, endPeriod);
}
```

Alternative defensive fix — always clamp when delegation has ended:

```solidity
if (endPeriod != type(uint32).max && endPeriod < currentValidatorPeriod) {
    // Clamp lastClaimablePeriod to never exceed endPeriod
    uint32 lastClaimable = endPeriod < completedPeriods ? endPeriod : completedPeriods;
    return (nextClaimablePeriod, lastClaimable);
}
```

Why these fixes work:

1. `>=` handles the equality case so the final legitimate period is returned as the last claimable period.
2. The defensive fix ensures that when a delegation has ended, the last claimable period never exceeds the delegation `endPeriod`.

## Notes

* All links and file references in this page are preserved as in the original report.
