# 59100 sc low periodattimestamp returns current period instead of queried historical period

**Submitted on Nov 8th 2025 at 17:03:24 UTC by @jayx for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59100
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `periodAtTimestamp(uint48 timestamp)` function in FirelightVault is advertised to return the period number corresponding to a given timestamp parameter, enabling historical period queries as part of the vault's "Historical tracking" feature. However, due to an implementation error in the helper function `_sinceEpoch()`, the function ignores the `timestamp` parameter and always calculates the period based on the current block timestamp (`Time.timestamp()`), effectively returning the current period for any input. This breaks the entire historical tracking feature for period-based queries, preventing users, external contracts, and off-chain systems from determining which period a historical timestamp belonged to, thereby rendering period-correlated historical data analysis impossible and causing operational failures in UIs, keepers, and analytics systems.

## Vulnerability Details

The root cause lies in the interaction between `periodAtTimestamp()` and its helper function `_sinceEpoch()`:

```solidity
/**
 * @notice Returns the period number for the timestamp given.
 * @dev Return value may be unreliable if period number given is far away in the future
 * @dev given that new period configurations can be added after nextPeriodEnd().
 * @return The period number corresponding to the given timestamp.
 */
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    // BUG: Uses _sinceEpoch() which calls Time.timestamp(), not the parameter 'timestamp'
    return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}

function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;  // ← Always uses current time, never query timestamp
}
```

The function correctly retrieves the `periodConfiguration` for the queried timestamp via `periodConfigurationAtTimestamp(timestamp)`, but then calculates the period offset using `_sinceEpoch(periodConfiguration.epoch)`, which internally calls `Time.timestamp() - epoch` instead of using `timestamp - epoch`.

**Expected behavior**:

```solidity
return periodConfiguration.startingPeriod + (timestamp - periodConfiguration.epoch) / periodConfiguration.duration;
```

**Actual behavior**:

```solidity
return periodConfiguration.startingPeriod + (Time.timestamp() - periodConfiguration.epoch) / periodConfiguration.duration;
```

This causes the function to always return the **current period**, regardless of whether the queried timestamp is in the past, present, or future.

## Impact Details

### Primary Impact: Contract Fails to Deliver Promised Returns

The FirelightVault audit documentation explicitly lists **"Historical tracking"** as a core feature. The `periodAtTimestamp()` function is part of this advertised feature set but is completely non-functional for historical queries, constituting a failure to deliver promised functionality.

### Concrete Consequences

**1. Historical Balance-to-Period Correlation Broken**

The vault provides `balanceOfAt(address, timestamp)` to query historical balances, but there is no way to determine which period that balance belonged to:

```solidity
// User queries historical state
uint256 historicalBalance = vault.balanceOfAt(user, pastTimestamp);  // ✅ Works
uint256 period = vault.periodAtTimestamp(pastTimestamp);              // ❌ Returns current period

// Consequence: Cannot correlate historical balances with periods
uint256 withdrawals = vault.withdrawalsOf(period, user);  // Wrong period queried!
```

Any logic attempting to correlate historical balances with period-based data (such as pending withdrawals) will query the wrong period.

## References

<https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L249C8-L249C123>

## Proof of Concept

## Proof of Concept

```javascript
it('BUG: periodAtTimestamp(nextPeriodStart) should be current+1 but equals current (ignores ts)', async () => {
  // Normalize everything to BigInt
  const curr = BigInt(await firelight_vault.currentPeriod());
  const currEnd = BigInt(await firelight_vault.currentPeriodEnd());
  const nextStart = currEnd + 1n;

  // Sanity: nextStart is within the current configuration window
  // Call the function under test with a valid future timestamp
  const got = BigInt(await firelight_vault.periodAtTimestamp(nextStart));

  // Characterize current buggy behavior: it returns the current period
  expect(got).to.equal(curr);

  // Optional: include a failing assertion for intended behavior if you want a red test
  // expect(got).to.equal(curr + 1n); // uncomment when fixing the contract
});

```
