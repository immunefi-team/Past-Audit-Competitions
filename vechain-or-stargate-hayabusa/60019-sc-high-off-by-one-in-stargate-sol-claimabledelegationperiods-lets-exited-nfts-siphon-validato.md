# 60019 sc high off by one in stargate sol claimabledelegationperiods lets exited nfts siphon validator rewards leading to protocol insolvency

**Submitted on Nov 17th 2025 at 15:40:21 UTC by @ElouJoe for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60019
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Protocol insolvency
  * Theft of unclaimed yield

## Description

### Brief/Intro

An off-by-one in Stargate.sol’s `_claimableDelegationPeriods` allows an exited NFT to keep claiming future validator rewards after its delegation ended, diverting other delegators’ VTHO and ultimately draining the reward pool to insolvency.

### Vulnerability Details

When an exit is requested, `endPeriod` is set. After the user claims through the true `endPeriod`, `lastClaimedPeriod = endPeriod`, so `nextClaimablePeriod` becomes `endPeriod + 1` and the check `endPeriod > nextClaimablePeriod` fails. The code falls back to `(nextClaimablePeriod, completedPeriods)` and treats the NFT as still active. This lets post-exit periods be claimed indefinitely.

Affected code: `_claimableDelegationPeriods` in packages/contracts/contracts/Stargate.sol.

### Impact Details

* Theft of other delegators’ unclaimed yield each completed period (misrouted to the exited NFT).
* Persistent drain escalates to Critical: Protocol insolvency as the reward stream is continuously siphoned.

## References

* Source: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* In-repo PoC: packages/contracts/test/unit/Stargate/DelegationBugGhostRewards.test.ts\
  Run: yarn hardhat test --network hardhat test/unit/Stargate/DelegationBugGhostRewards.test.ts

## Link to Proof of Concept

<https://gist.github.com/eloufirjawad/283c7077fdccd6eee3953ff4125a5be6>

## Proof of Concept

{% stepper %}
{% step %}

### Step 1

Stake and delegate an NFT (attacker) to a valid validator; also stake and delegate a second NFT (honest user) so the validator has a non-zero denominator.
{% endstep %}

{% step %}

### Step 2

Advance validator completed periods until the delegation is ACTIVE and claimable.
{% endstep %}

{% step %}

### Step 3

Call `requestDelegationExit(attackerTokenId)`; advance until exit finalizes (`endPeriod` set and `currentValidatorPeriod > endPeriod`).
{% endstep %}

{% step %}

### Step 4

Claim once to bring `lastClaimedPeriod` up to `endPeriod` (this makes `nextClaimablePeriod = endPeriod + 1`).
{% endstep %}

{% step %}

### Step 5

Advance one or more periods; observe `claimableRewards(attackerTokenId) > 0` even though status is EXITED, because `endPeriod > nextClaimablePeriod` is false and the function falls back to `(nextClaimablePeriod, completedPeriods)`.
{% endstep %}

{% step %}

### Step 6

Call `claimRewards(attackerTokenId)`; attacker receives VTHO for post‑exit periods. Repeat steps 5–6 indefinitely.
{% endstep %}
{% endstepper %}

What proves the bug

* After claiming through `endPeriod`, `claimableRewards` remains positive for each new completed period and `claimRewards` continues paying the exited NFT.
* Root cause: once `lastClaimedPeriod == endPeriod`, `nextClaimablePeriod = endPeriod + 1`, making `endPeriod > nextClaimablePeriod` false; `_claimableDelegationPeriods` falls back to `(nextClaimablePeriod, completedPeriods)`, treating the NFT as still active and enabling post‑exit claims.

## How to run the in-repo PoC

One-liner (copy/paste):

```bash
cd /Users/jawad/Documents/Python/Crypto/Immunefi/vechain/stargate-contracts-jose-update-contracts-to-hayabusa/packages/contracts && VITE_APP_ENV=local yarn hardhat test --network hardhat test/unit/Stargate/DelegationBugGhostRewards.test.ts
```

Two-step:

```bash
cd /Users/jawad/Documents/Python/Crypto/Immunefi/vechain/stargate-contracts-jose-update-contracts-to-hayabusa/packages/contracts

export VITE_APP_ENV=local

yarn hardhat test --network hardhat test/unit/Stargate/DelegationBugGhostRewards.test.ts
```

***

If you want, I can:

* Extract the exact vulnerable code snippet from the referenced repository and include a minimal annotated excerpt,
* Suggest a minimal patch (preserving original semantics) to fix the off-by-one logic.
