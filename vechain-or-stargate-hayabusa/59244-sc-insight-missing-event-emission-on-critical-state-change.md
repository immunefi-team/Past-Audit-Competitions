# 59244 sc insight missing event emission on critical state change

**Submitted on Nov 10th 2025 at 10:42:38 UTC by @akioniace for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59244
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>

## Description

### Brief / Intro

The contract `Stargate` does not emit an event upon a critical state change. This omission reduces on-chain transparency and makes it difficult for off-chain systems to track important changes. It also breaks the best practice of emitting events on important state changes.

### Vulnerability Details

The function `Stargate::setMaxClaimablePeriods` performs a critical state change by updating the maximum claimable period value, but it does not emit any event notifying about this change. Emitting an event on such state updates helps off-chain infrastructure (indexers, explorers, frontends) and improves transparency and auditability.

Relevant code:

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L943-L951>

```solidity
    /// @inheritdoc IStargate
    function setMaxClaimablePeriods(
        uint32 _maxClaimablePeriods
    ) external onlyRole(DEFAULT_ADMIN_ROLE) {
        if (_maxClaimablePeriods == 0) {
            revert InvalidMaxClaimablePeriods();
        }
        _getStargateStorage().maxClaimablePeriods = _maxClaimablePeriods;
        // @audit-issue: missing event emission
    }
```

## Impact Details

This is primarily a code-quality / best-practices issue. The consequences of missing event emission include:

* Reduced Transparency: Off-chain parties cannot easily verify important state changes.
* Poor Developer and User Experience: Front-ends and indexers cannot rely on an event to discover or react to the change without scanning on-chain state.
* Security Concerns: Absence of event logs makes it harder to audit and detect malicious or accidental changes.

## References

* <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L943-L951>

## Proof of Concept

{% stepper %}
{% step %}

### Step 1

Copy-paste the following test function into `packages/contracts/test/unit/Stargate/Rewards.test.ts`:

```js
it("should emit an event when max claimable periods is updated (currently missing)", async () => {
        // We expect the contract to emit something like "MaxClaimablePeriodsUpdated"
        // But since it does NOT, this test will fail — proving the missing event issue.

        await expect(
            stargateContract.connect(deployer).setMaxClaimablePeriods(2000)
        ).to.emit(stargateContract, "MaxClaimablePeriodsUpdated") // <-- just mocked event for showing 
          .withArgs(2000);
    });
```

{% endstep %}

{% step %}

### Step 2

Run the tests. The test will fail with an error indicating no event was emitted:

```
@repo/contracts:test:hardhat:   203 passing (1m)
@repo/contracts:test:hardhat:   1 failing
@repo/contracts:test:hardhat: 
@repo/contracts:test:hardhat:   1) shard-u4: Stargate: Rewards
@repo/contracts:test:hardhat:        should emit an event when max claimable periods is updated (currently missing):
@repo/contracts:test:hardhat:      AssertionError: Event "MaxClaimablePeriodsUpdated" doesn't exist in the contract
```

{% endstep %}
{% endstepper %}
