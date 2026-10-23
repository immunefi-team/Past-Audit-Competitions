# 59334 sc low periodattimestamp function uses current timestamp instead of input parameter causing incorrect period calculation for historical or future queries

## #59334 \[SC-Low] \`periodAtTimestamp\` function uses current timestamp instead of input parameter, causing incorrect period calculation for historical or future queries

**Submitted on Nov 11th 2025 at 08:43:14 UTC by @perseverance for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59334
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

### Description

## Short summary

The `periodAtTimestamp(uint48 timestamp)` function in `FirelightVault.sol` accepts a `timestamp` parameter but internally uses `Time.timestamp()` (current block timestamp) instead of the provided `timestamp` when calculating the period number. This causes the function to return incorrect period numbers when querying historical or future timestamps, breaking the expected behavior of the function.

While this bug does not directly cause loss of funds, it violates the contract's intended functionality and can lead to incorrect period calculations for historical data queries, which may affect off-chain integrations, analytics, or any system relying on accurate period information for past timestamps.

### Background Information

The `FirelightVault` contract implements a period-based withdrawal system where users can request withdrawals that become available in future periods. The contract maintains multiple period configurations, each with an epoch, duration, and starting period number.

The `periodAtTimestamp` function is designed to return the period number corresponding to a given timestamp. This is useful for:

* Querying historical period information
* Verifying period calculations for past events
* Off-chain systems that need to determine which period a transaction occurred in

#### Code Analysis

The bug is located in the `periodAtTimestamp` function:

<https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol#L246-L250>

```solidity
 	/**
     * @notice Returns the period number for the timestamp given.
     * @dev Return value may be unreliable if period number given is far away in the future
     * @dev given that new period configurations can be added after nextPeriodEnd().
     * @return The period number corresponding to the given timestamp.
     */
    function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
        PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        // solhint-disable-next-line max-line-length
        return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
    }

```

The function correctly retrieves the appropriate `periodConfiguration` for the given `timestamp` using `periodConfigurationAtTimestamp(timestamp)`. However, it then calls `_sinceEpoch(periodConfiguration.epoch)`, which uses the **current timestamp** instead of the provided input parameter `timestamp`:

<https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol#L795-L797>

```solidity
	function _sinceEpoch(uint48 epoch) private view returns (uint48) {
        return Time.timestamp() - epoch;
    }
```

#### Expected Behavior

The function should calculate the period number using the provided `timestamp`:

```
period = startingPeriod + (timestamp - epoch) / duration
```

#### Actual Behavior

The function incorrectly calculates the period number using the current block timestamp:

```
period = startingPeriod + (Time.timestamp() - epoch) / duration
```

This means:

* When querying a past timestamp, the function returns the period number for the current time, not the past timestamp
* When querying a future timestamp, the function returns the period number for the current time, not the future timestamp
* The function only returns correct results when querying with the current timestamp

#### Impact on Other Functions

The bug affects any code that relies on `periodAtTimestamp` for historical or future queries. However, the `currentPeriod()` function works correctly because it calls `periodAtTimestamp(Time.timestamp())`, which happens to work due to the bug using the same current timestamp:

```264:266:firelight/firelight-core/contracts/firelightvault.sol
function currentPeriod() public view returns (uint256) {
    return periodAtTimestamp(Time.timestamp());
}
```

### Severity Assessment

Bug Severity: **Low**

**Impact:**

* Contract fails to deliver promised returns, but doesn't lose value
* Function does not work as documented/intended

**Likelihood:**

* High - The bug affects all historical/future timestamp queries
* The function is public and can be called by anyone
* However, the impact is limited to informational queries and does not affect core functionality like deposits, withdrawals, or claims

**Justification for Low Severity:**

* No funds are at risk
* Core vault functionality (deposits, withdrawals, claims) is unaffected
* The bug only affects informational queries about historical periods

### Recommendation

Fix the `periodAtTimestamp` function to use the provided `timestamp` parameter instead of `Time.timestamp()`:

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    // Calculate time since epoch using the provided timestamp, not current time
    uint48 timeSinceEpoch = timestamp - periodConfiguration.epoch;
    return periodConfiguration.startingPeriod + timeSinceEpoch / periodConfiguration.duration;
}
```

### Link to Proof of Concept

<https://gist.github.com/Perseverancesuccess2021/4390a2d10d541b5ffc2a76b03e233619#file-periodattimestamp\\_bug-js>

### Proof of Concept

## Proof of Concept

The following test case demonstrates the bug:

Full Runnable POC: <https://gist.github.com/Perseverancesuccess2021/4390a2d10d541b5ffc2a76b03e233619#file-periodattimestamp\\_bug-js>

```javascript
// Test file: test/periodAtTimestamp_bug.js

it('should demonstrate bug: periodAtTimestamp uses current time instead of input timestamp', async () => {
    // Step 1: Get initial period configuration information
    const [initial_epoch, initial_duration, initial_starting_period] = await firelight_vault.periodConfigurations(0)
    const initial_timestamp = BigInt(await time.latest())
    
    console.log('Initial setup:')
    console.log(`  Epoch: ${initial_epoch}`)
    console.log(`  Duration: ${initial_duration}`)
    console.log(`  Starting Period: ${initial_starting_period}`)
    console.log(`  Initial Timestamp: ${initial_timestamp}`)

    // Step 2: Wait for a sufficient amount of time to create a clear difference between periods
    // Wait 1.5 periods to ensure there's a difference
    const time_to_advance = Number(initial_duration) * 1.5
    await time.increase(time_to_advance)
    
    // Past timestamp: in the middle of the first period (0.5 period after initial_timestamp)
    const time_offset = BigInt(Math.floor(Number(initial_duration) * 0.5))
    const timestamp_past = initial_timestamp + time_offset
    const current_timestamp = BigInt(await time.latest())
    
    console.log('\nAfter time advance:')
    console.log(`  Past Timestamp (query target): ${timestamp_past}`)
    console.log(`  Current Timestamp: ${current_timestamp}`)
    console.log(`  Time difference: Current Timestamp - Past Timestamp =  ${current_timestamp - timestamp_past} seconds`)

    // Step 3: Calculate the CORRECT period number for the past timestamp (manual calculation)
    // Correct formula: startingPeriod + (timestamp - epoch) / duration
    const time_since_epoch_correct = Number(timestamp_past) - Number(initial_epoch)
    const period_correct = Number(initial_starting_period) + Math.floor(time_since_epoch_correct / Number(initial_duration))
    
    console.log('\nCorrect calculation:')
    console.log(`  Time since epoch: ${time_since_epoch_correct} seconds`)
    console.log(`  Correct Period: ${period_correct}`)

    
    // Step 4: Call periodAtTimestamp function with the past timestamp
    const period_from_contract = await firelight_vault.periodAtTimestamp(timestamp_past)
    
    console.log('\nContract result:')
    console.log('\nWrong calculation (what contract does):')
    console.log(`  periodAtTimestamp(${timestamp_past}): ${period_from_contract}`)

       
    // Verify that the contract result does NOT equal the correct result
    expect(Number(period_from_contract)).to.not.equal(period_correct,
      'Bug confirmed: contract uses current time instead of input timestamp')
    
    console.log('\n✅ Bug confirmed!')
    console.log(`   Expected period for timestamp ${timestamp_past}: ${period_correct}`)
    console.log(`   Contract returned: ${period_from_contract}`)
    console.log(`   Contract is using current timestamp ${current_timestamp} instead of input ${timestamp_past}`)
  })
```

### Running the Test

Full POC: <https://gist.github.com/Perseverancesuccess2021/4390a2d10d541b5ffc2a76b03e233619#file-periodattimestamp\\_bug-js>

Copy the full POC into firelight-core/test

```bash
cd firelight-core
npx hardhat test test/periodAtTimestamp_bug.js
```

The test will pass, confirming that the contract returns incorrect values for historical timestamp queries.

Test log

```
npx hardhat test test/periodAtTimestamp_bug.js


  periodAtTimestamp Bug Test
Initial setup:
  Epoch: 1762846725
  Duration: 172800
  Starting Period: 0
  Initial Timestamp: 1762846726

After time advance:
  Past Timestamp (query target): 1762933126
  Current Timestamp: 1763105926
  Time difference: Current Timestamp - Past Timestamp =  172800 seconds

Correct calculation:
  Time since epoch: 86401 seconds
  Correct Period: 0

Contract result:

Wrong calculation (what contract does):
  periodAtTimestamp(1762933126): 1

✅ Bug confirmed!
   Expected period for timestamp 1762933126: 0
   Contract returned: 1
   Contract is using current timestamp 1763105926 instead of input 1762933126
    ✔ should demonstrate bug: periodAtTimestamp uses current time instead of input timestamp (45ms)
    ✔ should show correct behavior when querying current timestamp


  2 passing (5s)

```

### Explanation:

Query target timestamp = 1762933126

period duration = 172800 = 2 days

Time since epoch = 86401 = 1/2 period duration

\=> The period of that timestamp should be equal starting\_period = 0

But the contract returns 1 because it takes current timestamp in calculation.
