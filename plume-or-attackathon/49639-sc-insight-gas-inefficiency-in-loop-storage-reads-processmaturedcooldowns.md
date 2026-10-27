# 49639 sc insight gas inefficiency in loop storage reads processmaturedcooldowns

**Submitted on Jul 17th 2025 at 20:24:34 UTC by @Opzteam for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49639
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>

## Impacts

(As provided in the original report.)

## Description

The `_processMaturedCooldowns` function performs multiple storage reads within loops, creating gas inefficiencies for users with many validator associations.

Example snippet:

```solidity
// File: contracts-src/facets/StakingFacet.sol:854-871
for (uint256 i = 0; i < userAssociatedValidators.length; i++) {
    uint16 validatorId = userAssociatedValidators[i];
    PlumeStakingStorage.CooldownEntry memory cooldownEntry = $.userValidatorCooldowns[user][validatorId];
    // ... multiple storage reads follow
    if ($.userValidatorStakes[user][validatorId].staked == 0) {
        // ... more storage access
    }
}
```

* Batch load data before loop execution to minimize storage reads.
* Users with many validator associations face significantly higher gas costs during cooldown processing operations, potentially making the protocol prohibitively expensive for power users and creating scalability limitations.

## Proof of Concept

Implement batch loading by pre-fetching validator states and user stakes into memory arrays before loop execution, reducing per-iteration storage access from O(n) to O(1) for repeated data.

Example approach:

```solidity
// Batch load all data once
struct BatchData {
    uint256[] cooldownAmounts;
    bool[] validatorExists;
    bool[] validatorSlashed;
    uint256[] userStakes;
}

BatchData memory batch = _loadBatchData(user, userAssociatedValidators);

for (uint256 i = 0; i < userAssociatedValidators.length; i++) {
    // Use memory data instead of storage reads
    if (batch.cooldownAmounts[i] == 0) continue;
    
    bool canRecover = _canRecoverOptimized(batch.validatorExists[i], batch.validatorSlashed[i]);
    
    if (canRecover && batch.userStakes[i] == 0) {
        PlumeValidatorLogic.removeStakerFromValidator($, user, validatorId);
    }
}
```

Reported gas savings: Total: 50-70% gas reduction for multi-validator users.
