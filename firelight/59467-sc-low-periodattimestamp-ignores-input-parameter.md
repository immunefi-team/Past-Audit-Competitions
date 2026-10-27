# #59467 \[SC-Low] periodattimestamp ignores input parameter

**Submitted on Nov 12th 2025 at 15:12:47 UTC by @Orionn for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59467
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `periodAtTimestamp()` view function in FirelightVault.sol completely ignores its input timestamp parameter and always returns the current period instead. This logic error makes historical period queries impossible, breaking analytics systems, off-chain indexers, and any protocol attempting to determine which period a past transaction occurred in. The function's timestamp parameter is entirely non-functional.

## Vulnerability Details

**Location:** `contracts/FirelightVault.sol` - `periodAtTimestamp()` function (line 269) and `_sinceEpoch()` helper (line 916)

**Root Cause:**

The `periodAtTimestamp()` function is supposed to calculate which period a given timestamp falls into, but it incorrectly uses the CURRENT timestamp instead of the input parameter:

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory pc = periodConfigurationAtTimestamp(timestamp);
    // @audit _sinceEpoch uses current time
    return pc.startingPeriod + _sinceEpoch(pc.epoch) / pc.duration;
    //                          ^^^^^^^^^^^^^^^^^ BUG: Uses current time
}

function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;  // ← Uses CURRENT time, ignores input!
}
```

**The Bug:**

* `periodAtTimestamp(timestamp)` receives a timestamp parameter
* It calls `_sinceEpoch(pc.epoch)` which should calculate `timestamp - epoch`
* But `_sinceEpoch()` uses `Time.timestamp()` (current time) instead
* Result: The function **always** returns the current period, regardless of input

## Impact Details

### 1. **Historical Analytics Completely Broken**

Off-chain systems cannot determine which period past transactions occurred in:

```javascript
// Transaction happened at period 3
const txTimestamp = 1764688864

// Query returns period 7 (current), not 3 (actual)
const period = await vault.periodAtTimestamp(txTimestamp) // WRONG!
```

**Impact:**

* **Dashboards** show incorrect period numbers for historical transactions
* **Charts/graphs** plotting period-based data are meaningless
* **Reports** analyzing period performance use wrong data

### 2. **Indexer/Subgraph Integration Failures**

Blockchain indexers (The Graph, etc.) rely on this function to categorize historical events:

```graphql

# Indexer tries to determine which period a Deposit event occurred in
query {
  deposit(id: "0x123...") {
    timestamp: 1764688864
    period: periodAtTimestamp(1764688864)  # Returns wrong period!
  }
}
```

**Impact:**

* **Event indexing** assigns wrong periods to all historical events
* **Period-based queries** return incorrect results
* **Time-series data** is corrupted
* **Historical snapshots** are impossible to reconstruct

### 3. **Integration Protocol Issues**

Protocols integrating with FirelightVault cannot analyze historical behavior:

**Example Use Case:**

```javascript
// Protocol wants to analyze withdrawal patterns per period
async function analyzeWithdrawalHistory(withdrawalEvents) {
    for (const event of withdrawalEvents) {
        // This returns current period, not the actual period!
        const period = await vault.periodAtTimestamp(event.timestamp)
        
        // All events appear to be in current period
        periodStats[period]++  // Data is completely wrong
    }
}
```

**Impact:**

* **Risk analysis** of period-based withdrawal patterns fails
* **Trend detection** is impossible
* **Historical comparisons** are meaningless
* **Integration testing** based on historical data doesn't work

### 4. **User-Facing Analytics Broken**

Users cannot view their historical activity correctly:

```javascript
// User's transaction history page
"Your deposits by period:"
- Period 7: 10 deposits  ← WRONG (all historical deposits show as period 7)
- Period 6: 0 deposits
- Period 5: 0 deposits
```

**Impact:**

* **User dashboards** show all historical activity in current period
* **Personal analytics** are meaningless
* **Transaction history** cannot be properly categorized
* **Tax reporting** based on periods is incorrect

### 5. **Function is Completely Useless**

The function's entire purpose is defeated:

```solidity
// Function signature promises to return period for given timestamp
function periodAtTimestamp(uint48 timestamp) public view returns (uint256)
```

**Reality:** The `timestamp` parameter is completely ignored. This is equivalent to:

```solidity
// What it actually does
function periodAtTimestamp(uint48 /* ignored */) public view returns (uint256) {
    return currentPeriod();  // Just returns current period
}
```

## References

* **Affected Contract:** `https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol`
* **Affected Function:** `periodAtTimestamp()` (line 269)
* **Helper Function:** `_sinceEpoch()` (line 916)

## Proof of Concept

## Proof of Concept

**Test File:** `test/periodAtTimestamp_poc.js`

**Test Details:**

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')

describe('periodAtTimestamp Uses Wrong Timestamp', function () {
  const PERIOD_DURATION = 604800 // 1 week

  let firelight_vault, users, utils, config

  before(async () => {
    ({ firelight_vault, users, utils, config } = await loadFixture(deployVault.bind()))
    const DEPOSIT_AMOUNT = ethers.parseUnits('10000', 6)
    await Promise.all(users.map(account => utils.mintAndApprove(DEPOSIT_AMOUNT, account)))
  })

  it('periodAtTimestamp always returns current period (ignores input)', async () => {
    // Setup: Create some time gap between periods
    await time.increase(PERIOD_DURATION * 3)

    const currentTime = await time.latest()
    const currentPeriod = await firelight_vault.currentPeriod()

    console.log(`\n   Current time: ${currentTime}`)
    console.log(`   Current period: ${currentPeriod}`)

    // Query with PAST timestamps - should return earlier periods
    const pastTime1 = currentTime - PERIOD_DURATION * 3
    const pastTime2 = currentTime - PERIOD_DURATION * 2
    const pastTime3 = currentTime - PERIOD_DURATION * 1

    const period1 = await firelight_vault.periodAtTimestamp(pastTime1)
    const period2 = await firelight_vault.periodAtTimestamp(pastTime2)
    const period3 = await firelight_vault.periodAtTimestamp(pastTime3)

    console.log(`\n   Query past timestamp (3 weeks ago): ${pastTime1}`)
    console.log(`   Returned period: ${period1}`)
    console.log(`   Expected: 1 or 2 (early period)`)

    console.log(`\n   Query past timestamp (2 weeks ago): ${pastTime2}`)
    console.log(`   Returned period: ${period2}`)

    console.log(`\n   Query past timestamp (1 week ago): ${pastTime3}`)
    console.log(`   Returned period: ${period3}`)

    // BUG: All queries return the SAME period (current period)
    console.log(`\n   All returned periods: [${period1}, ${period2}, ${period3}]`)
    console.log(`   Current period: ${currentPeriod}`)
    console.log(`   All equal to current? ${period1 === currentPeriod && period2 === currentPeriod && period3 === currentPeriod}\n`)

    // All should return current period due to bug
    expect(period1).to.equal(currentPeriod) // Should be earlier period
    expect(period2).to.equal(currentPeriod) // Should be earlier period
    expect(period3).to.equal(currentPeriod) // Should be earlier period
  })

  it('demonstrates the function completely ignores timestamp parameter', async () => {
    const currentPeriod = await firelight_vault.currentPeriod()
    const currentTime = await time.latest()

    // Try different PAST timestamps (within valid range)
    const timestamps = [
      currentTime - PERIOD_DURATION * 2,   // 2 weeks ago
      currentTime - PERIOD_DURATION * 1,   // 1 week ago
      currentTime - 100,                    // 100 seconds ago
      currentTime                           // now
    ]

    const periods = []
    for (const ts of timestamps) {
      const period = await firelight_vault.periodAtTimestamp(ts)
      periods.push(period)
    }

    console.log('   Different timestamps queried:')
    timestamps.forEach((ts, i) => {
      const diff = (Number(currentTime) - Number(ts)) / PERIOD_DURATION
      console.log(`   [${i}] ${diff.toFixed(3)} weeks ago: period ${periods[i]}`)
    })

    console.log(`\n   All periods identical: ${periods.every(p => p === periods[0])}`)
    console.log(`   All equal current period: ${periods.every(p => p === currentPeriod)}\n`)

    // BUG: All return the same period (current period)
    expect(periods[0]).to.equal(currentPeriod) // Should be earlier
    expect(periods[1]).to.equal(currentPeriod) // Should be earlier  
    expect(periods[2]).to.equal(currentPeriod) // Should be same (close to current)
    expect(periods[3]).to.equal(currentPeriod) // This one is correct

    // All should be identical due to bug
    expect(new Set(periods).size).to.equal(1)
  })

  it('shows impact on historical event analysis', async () => {
    const user = users[0]

    // Simulate historical events at different times
    const events = []

    // Event 1: Deposit at period 0
    await firelight_vault.connect(user).deposit(ethers.parseUnits('1000', 6), user.address)
    const event1Time = await time.latest()
    const event1Period = await firelight_vault.currentPeriod()
    events.push({ name: 'Deposit', time: event1Time, actualPeriod: event1Period })

    // Advance time to period 1
    await time.increase(PERIOD_DURATION * 2)

    // Event 2: Withdrawal request at period 1+
    await firelight_vault.connect(user).redeem(ethers.parseUnits('500', 6), user.address, user.address)
    const event2Time = await time.latest()
    const event2Period = await firelight_vault.currentPeriod()
    events.push({ name: 'Withdraw', time: event2Time, actualPeriod: event2Period })

    // Advance time to period 2+
    await time.increase(PERIOD_DURATION * 2)

    // Now query historical periods for those events
    console.log('   Historical Event Analysis:')
    for (const event of events) {
      const queriedPeriod = await firelight_vault.periodAtTimestamp(event.time)
      const correct = queriedPeriod === event.actualPeriod

      console.log(`   ${event.name}:`)
      console.log(`     Actual period when occurred: ${event.actualPeriod}`)
      console.log(`     Query periodAtTimestamp(${event.time}): ${queriedPeriod}`)
      console.log(`     Correct? ${correct ? '✓' : '✗ WRONG'}`)

      // BUG: Queried period will be current period, not historical period
      expect(queriedPeriod).to.not.equal(event.actualPeriod)
    }

    const currentPeriod = await firelight_vault.currentPeriod()
    console.log(`\n   Current period: ${currentPeriod}`)
    console.log(`   Both queries returned: ${currentPeriod} (current period, not historical)\n`)
  })
})
```

Run the PoC:

```bash
npx hardhat test test/periodAtTimestamp_poc.js
```

**Expected Output:**

```bash
  periodAtTimestamp Uses Wrong Timestamp

   Current time: 1764770884
   Current period: 3

   Query past timestamp (3 weeks ago): 1762956484
   Returned period: 3
   Expected: 1 or 2 (early period)

   Query past timestamp (2 weeks ago): 1763561284
   Returned period: 3

   Query past timestamp (1 week ago): 1764166084
   Returned period: 3

   All returned periods: [3, 3, 3]
   Current period: 3
   All equal to current? true

    ✔ periodAtTimestamp always returns current period (ignores input)
   Different timestamps queried:
   [0] 2.000 weeks ago: period 3
   [1] 1.000 weeks ago: period 3
   [2] 0.000 weeks ago: period 3
   [3] 0.000 weeks ago: period 3

   All periods identical: true
   All equal current period: true

    ✔ demonstrates the function completely ignores timestamp parameter
   Historical Event Analysis:
   Deposit:
     Actual period when occurred: 3
     Query periodAtTimestamp(1764770885): 7
     Correct? ✗ WRONG
   Withdraw:
     Actual period when occurred: 5
     Query periodAtTimestamp(1765980486): 7
     Correct? ✗ WRONG

   Current period: 7
   Both queries returned: 7 (current period, not historical)

    ✔ shows impact on historical event analysis (2970ms)


  3 passing (8s)
```
