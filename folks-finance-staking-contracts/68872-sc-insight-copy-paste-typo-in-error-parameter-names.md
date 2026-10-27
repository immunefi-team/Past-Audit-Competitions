# 68872 sc insight copy paste typo in error parameter names

**Submitted on Mar 11th 2026 at 18:35:20 UTC by @ZenHunter for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #68872
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/interfaces/IStakingV1.sol>
* **Impacts:**

## Description

### Brief/Intro

Two error parameters in `IStakingV1` are named with an `Apr` suffix but hold a duration in seconds. The suffix was copied from the adjacent `StakingPeriodAprDiffer` error. On-chain behaviour is unaffected, but off-chain tools decoding these errors will label a duration field as an APR value, producing misleading output when debugging a failed `stake()` call.

### Vulnerability Details

The affected definitions are `IStakingV1.sol#L64–L69`:

```solidity
// src/IStakingV1.sol#L64-L69
error StakingPeriodStakingDurationDiffer(
    uint8 periodIndex, uint64 expectedMaxStakingDurationApr, uint64 periodStakingDuration
);
error StakingPeriodUnlockDurationDiffer(
    uint8 periodIndex, uint64 expectedMaxUnlockDurationApr, uint64 periodUnlockDuration
);
```

The call sites in `Staking.sol` confirm what each parameter actually holds:

```solidity
// src/Staking.sol#L250-L257
revert StakingPeriodStakingDurationDiffer(
    periodIndex, params.maxStakingDurationSeconds, stakingPeriod.stakingDurationSeconds
);
// ...
revert StakingPeriodUnlockDurationDiffer(
    periodIndex, params.maxUnlockDurationSeconds, stakingPeriod.unlockDurationSeconds
);
```

`params.maxStakingDurationSeconds` and `params.maxUnlockDurationSeconds` are slippage-protection bounds on staking duration in seconds — not APR values. The `Apr` suffix was copied from `StakingPeriodAprDiffer`, the error defined immediately below these two.

### Impact Details

**Impact category:** Documentation Improvements

### References

* Affected lines: `IStakingV1.sol#L64–L69`
* Call sites confirming parameter semantics: `Staking.sol#L250–L257`

## Recommendation

Rename the two parameters to match their actual semantics:

```solidity
error StakingPeriodStakingDurationDiffer(
    uint8 periodIndex, uint64 expectedMaxStakingDuration, uint64 periodStakingDuration
);
error StakingPeriodUnlockDurationDiffer(
    uint8 periodIndex, uint64 expectedMaxUnlockDuration, uint64 periodUnlockDuration
);
```

No ABI selector changes — callers and existing integrations are unaffected.

## Proof of Concept

No executable PoC is required. The mismatch is directly visible by comparing the error definition with its call site:

```solidity
// Definition (IStakingV1.sol#L64–L66) — parameter name says "Apr"
error StakingPeriodStakingDurationDiffer(
    uint8 periodIndex, uint64 expectedMaxStakingDurationApr, uint64 periodStakingDuration
);

// Call site (Staking.sol#L250–L252) — argument is a duration in seconds, not an APR
revert StakingPeriodStakingDurationDiffer(
    periodIndex, params.maxStakingDurationSeconds, stakingPeriod.stakingDurationSeconds
);

// Definition (IStakingV1.sol#L67–L69) — parameter name says "Apr"
error StakingPeriodUnlockDurationDiffer(
    uint8 periodIndex, uint64 expectedMaxUnlockDurationApr, uint64 periodUnlockDuration
);

// Call site (Staking.sol#L255–L257) — argument is a duration in seconds, not an APR
revert StakingPeriodUnlockDurationDiffer(
    periodIndex, params.maxUnlockDurationSeconds, stakingPeriod.unlockDurationSeconds
);
```

`params.maxStakingDurationSeconds` is declared as `uint64 maxStakingDurationSeconds` in `StakeParams` (`IStakingV1.sol#L30`) — a duration measured in seconds, not a rate in basis points.
