# #59879 \[SC-Low] logic bug in periodattimestamp

**Submitted on Nov 16th 2025 at 16:59:32 UTC by @Immanux2160 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59879
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `periodAtTimestamp` function in the `FirelightVault` contract contains a logic flaw where it incorrectly uses the current `block.timestamp` instead of the provided timestamp parameter when calculating historical periods. This vulnerability completely breaks all historical data queries in the vault, if in production, this would render all historical audit trails unreliable, break any integrations relying on historical data, undermine user confidence in the protocol's accounting.

## Vulnerability Details

The bug is in the `periodAtTimestamp` function which is intended to return the period number for any given historical timestamp. However, the function incorrectly uses the current `block.timestamp` instead of the parameter timestamp when calculating time elapsed since the epoch.

```
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
        PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        // solhint-disable-next-line max-line-length
        //@audit it uses the current block.timestamp in _sinceEpoch calculation intead of the timestamp given in the parameter
        return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
    }
```

```
    function _sinceEpoch(uint48 epoch) private view returns (uint48) {
        return Time.timestamp() - epoch;//@audit-issue always uses current timestamp
    }

```

The problem: `_sinceEpoch(epoch)` always returns `block.timestamp - epoch` regardless of the timestamp parameter, this means `periodAtTimestamp(anyTimestamp)` always returns the current period number, historical queries become meaningless as they all return current state.

The function also always returns the period of "now," even if you're calculating for an arbitrary timestamp.

## Impact Details

The issue with `periodAtTimestamp` is that it ignores the timestamp passed into the function and instead uses the current block time when calculating the period. This means it returns the current period even when you ask for the period of a past or future timestamp. While it doesn’t cause direct loss of funds, it breaks the core assumption that periods can be computed deterministically from any timestamp.

Also this leads to inconsistencies between on-chain state and off-chain systems that rely on accurate period calculations such as frontends, indexers, or analytic tools that map historical balances (balanceOfAt) or withdrawals to specific periods.

## References

<https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol?#L245>

<https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol?#L795>

## Proof of Concept

## Proof of Concept

```
// const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
// const { deployVault } = require('./setup/fixtures.js')
// const { expect } = require('chai')

// /**
//  * Simple Proof: periodAtTimestamp is Broken
//  * 
//  * The Bug: periodAtTimestamp always returns the current period, 
//  * no matter what timestamp you ask about.
//  */
// describe('The periodAtTimestamp Bug', function () {
//   let firelight_vault

//   before(async () => {
//     const deployment = await loadFixture(deployVault.bind())
//     firelight_vault = deployment.firelight_vault
//   })

//   it('shows the bug with safe, valid timestamps', async () => {
//     console.log('Testing periodAtTimestamp with valid timestamps only\n')

//     // Get current time and period
//     const currentTime = await time.latest()
//     const currentPeriod = await firelight_vault.currentPeriod()
//     console.log(`We start in period ${currentPeriod} at time ${currentTime}`)

//     // Get the current period configuration to know valid timestamp range
//     const config = await firelight_vault.periodConfigurationAtTimestamp(currentTime)
//     console.log(`Current period started at ${config.epoch} and lasts ${config.duration} seconds`)

//     // Create safe timestamps within the current period
//     const periodStart = Number(config.epoch)
//     const safeTimestamps = [
//       periodStart + 3600,                    // 1 hour after period start
//       periodStart + Number(config.duration) / 2,  // Middle of period
//       currentTime                           // Current time
//     ]

//     console.log('\nTesting with safe timestamps:')
//     for (const timestamp of safeTimestamps) {
//       const result = await firelight_vault.periodAtTimestamp(timestamp)
//       console.log(`  Asking about time ${timestamp} → Period ${result}`)
//       expect(result).to.equal(currentPeriod) // All return current period (correct for now)
//     }
//     console.log('All return current period (expected, since we\'re in the same period)\n')

//     // Now let time pass to change periods
//     await time.increase(Number(config.duration) + 3600) // Move to next period
//     const newPeriod = await firelight_vault.currentPeriod()
//     console.log(`Time passed... we're now in period ${newPeriod}`)

//     // THE BUG: Ask about the same safe timestamps again
//     console.log(' BUG DEMONSTRATION:')
//     console.log('Asking about the same historical timestamps again:')
    
//     for (const timestamp of safeTimestamps) {
//       const result = await firelight_vault.periodAtTimestamp(timestamp)
//       console.log(`  Asking about time ${timestamp} → Period ${result}`)
      
//       // This is the bug: historical timestamps return current period
//       expect(result).to.equal(newPeriod)
//       expect(result).to.not.equal(currentPeriod)
//     }

//     console.log('\n BUG CONFIRMED:')
//     console.log(`Historical timestamps from period ${currentPeriod}`)
//     console.log(`Now return period ${newPeriod} (the current period)`)
//     console.log('The function uses current time instead of the timestamp parameter!')
//   })

//   it('proves the bug with a simple example', async () => {
//     console.log('Simple example to prove the bug:\n')

//     // Record current state
//     const timestamp = await time.latest()
//     const periodAtThatTime = await firelight_vault.currentPeriod()
//     console.log(`At time ${timestamp}, we're in period ${periodAtThatTime}`)

//     // Move to next period
//     const config = await firelight_vault.periodConfigurationAtTimestamp(timestamp)
//     await time.increase(Number(config.duration) + 3600)
    
//     const newPeriod = await firelight_vault.currentPeriod()
//     console.log(`Time moved forward... now in period ${newPeriod}`)

//     // Ask about the old timestamp
//     const result = await firelight_vault.periodAtTimestamp(timestamp)
//     console.log(`\nAsking: "What period was it at time ${timestamp}?"`)
//     console.log(`Should answer: ${periodAtThatTime}`)
//     console.log(`Actually answers: ${result}`)

//     console.log('PROOF:')
//     console.log(`It returns the current period (${newPeriod}) instead of the historical one (${periodAtThatTime})`)
    
//     expect(result).to.equal(newPeriod)
//   })

//   it('explains the problem simply', async () => {
//     console.log('\ Why this is a problem:\n')
    
//     console.log('The periodAtTimestamp function is supposed to answer:')
//     console.log('  "What period was it at this specific time?"')
//     console.log('')
//     console.log('But due to the bug, it actually answers:')
//     console.log('  "What period is it right now?"')
//     console.log('')
//     console.log('This makes historical period queries impossible.')
//     console.log('You can only get the current period, never past periods.')
//   })
// })

const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')

/**
 * POC: periodAtTimestamp wrongly shows an incorrect data
 *
 * Summary:
 *   periodAtTimestamp should tell us the period at ANY timestamp (past, present, future).
 *   Instead, it always returns the *current* period, completely ignoring the timestamp passed in.
 */

describe('periodAtTimestamp', function () {
  let firelight_vault

  before(async () => {
    const deployment = await loadFixture(deployVault.bind())
    firelight_vault = deployment.firelight_vault
  })

  it('demonstrates the bug using safe timestamps', async () => {
    console.log(`\n Checking periodAtTimestamp with timestamps that should be valid \n`)

    // Grab the current blockchain time and current period
    const now = await time.latest()
    const currentPeriod = await firelight_vault.currentPeriod()
    console.log(`Current period is ${currentPeriod}, at time ${now}`)

    // ask the vault for the period config at the current time
    const config = await firelight_vault.periodConfigurationAtTimestamp(now)
    console.log(
      `According to the vault, this period started at ${config.epoch} and lasts ${config.duration} seconds.`
    )

    // We'll pick a few timestamps safely inside this period
    const periodStart = Number(config.epoch)
    const safeTimestamps = [
      periodStart + 3600,                        // 1 hour after the period starts
      periodStart + Number(config.duration) / 2, // middle of the period
      now                                        // right now
    ]

    console.log(`\nTesting known-valid timestamps inside the same period:\n`)
    for (const ts of safeTimestamps) {
      const result = await firelight_vault.periodAtTimestamp(ts)
      console.log(`  Asking about timestamp ${ts} → Returned period ${result}`)
      expect(result).to.equal(currentPeriod)
    }

    console.log(
      `\nSo far, everything behaves correctly timestamps inside the same window map to the same period.\n`
    )

    // Now move time forward into the *next* period
    await time.increase(Number(config.duration) + 3600)
    const newPeriod = await firelight_vault.currentPeriod()
    console.log(`time passes... we Have advanced into a new period: ${newPeriod}\n`)

    console.log(`Now we repeat the same queries this is where the bug shows itself:\n`)
    for (const ts of safeTimestamps) {
      const result = await firelight_vault.periodAtTimestamp(ts)
      console.log(`  Asking again about timestamp ${ts} → Returned period ${result}`)

      // This is the core bug: older timestamps now incorrectly map to the new period
      expect(result).to.equal(newPeriod)
      expect(result).to.not.equal(currentPeriod)
    }

    console.log(`the bug explained`)
    console.log(`Historical timestamps from the previous period (${currentPeriod})`)
    console.log(`are now incorrectly reported as belonging to period ${newPeriod}.`)
    console.log(`The function is clearly using the *current* block timestamp internally.\n`)
  })

  it('shows a minimal reproduction of the bug', async () => {
    console.log(`\n Minimal Reproduction Example \n`)

    const t0 = await time.latest()
    const periodAtT0 = await firelight_vault.currentPeriod()

    console.log(`At time ${t0}, the vault says we're in period ${periodAtT0}.`)

    // Move time far enough ahead to roll into the next period
    const config = await firelight_vault.periodConfigurationAtTimestamp(t0)
    await time.increase(Number(config.duration) + 3600)

    const newPeriod = await firelight_vault.currentPeriod()
    console.log(`Time moves forward... now we're in period ${newPeriod}.`)

    const result = await firelight_vault.periodAtTimestamp(t0)

    console.log(`\nAsking again: “What period was it at time ${t0}?”`)
    console.log(`Expected answer: ${periodAtT0}`)
    console.log(`Actual answer:   ${result}`)

    console.log(`\n→ This confirms that periodAtTimestamp ignores the provided timestamp.`)

    expect(result).to.equal(newPeriod)
  })

  it('simplified explanation', async () => {
    console.log(`explanation of this bug`)
    console.log(`periodAtTimestamp is supposed to answer:`)
    console.log(`  “Given a timestamp, what period did that time fall into?”\n`)

    console.log(`But right now, the function really answers:`)
    console.log(`  “What period is it *right now*?”\n`)

    console.log(`This makes it impossible to query historical state.`)
    console.log(`Any feature relying on historical periods (snapshots, rewards, vesting, etc.)`)
    console.log(`will be incorrect because the function can never return past periods.\n`)
  })
})

```

```
POC output:

  periodAtTimestamp

 Checking periodAtTimestamp with timestamps that should be valid

Current period is 0, at time 1763311466
According to the vault, this period started at 1763311465 and lasts 604800 seconds.

Testing known-valid timestamps inside the same period:

  Asking about timestamp 1763315065 → Returned period 0
  Asking about timestamp 1763613865 → Returned period 0
  Asking about timestamp 1763311466 → Returned period 0

So far, everything behaves correctly timestamps inside the same window map to the same period.        

time passes... we Have advanced into a new period: 1

Now we repeat the same queries this is where the bug shows itself:

  Asking again about timestamp 1763315065 → Returned period 1
  Asking again about timestamp 1763613865 → Returned period 1
  Asking again about timestamp 1763311466 → Returned period 1
the bug explained
Historical timestamps from the previous period (0)
are now incorrectly reported as belonging to period 1.
The function is clearly using the *current* block timestamp internally.

    ✔ demonstrates the bug using safe timestamps (95ms)

 Minimal Reproduction Example

At time 1763919866, the vault says we're in period 1.
Time moves forward... now we're in period 2.

Asking again: “What period was it at time 1763919866?”
Expected answer: 1
Actual answer:   2

→ This confirms that periodAtTimestamp ignores the provided timestamp.
    ✔ shows a minimal reproduction of the bug (52ms)
explanation of this bug
periodAtTimestamp is supposed to answer:
  “Given a timestamp, what period did that time fall into?”

But right now, the function really answers:
  “What period is it *right now*?”

This makes it impossible to query historical state.
Any feature relying on historical periods (snapshots, rewards, vesting, etc.)
will be incorrect because the function can never return past periods.

    ✔ simplified explanation
```

Mitigation: use the timestamp as a parameter in `_sinceEpoch internal` function or change the function like this:

```
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    
    // possible solution: Use the parameter timestamp, not current block.timestamp
    uint48 timeSinceEpoch = timestamp - periodConfiguration.epoch;
    
    return periodConfiguration.startingPeriod + timeSinceEpoch / periodConfiguration.duration;
}
```

or change `_sinceEpoch` internal function to this:

```
function _sinceEpoch(uint48 epoch, uint48 timestamp) private pure returns (uint48) {
    return timestamp - epoch;
}
```
