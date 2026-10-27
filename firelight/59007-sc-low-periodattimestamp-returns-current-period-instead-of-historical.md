# #59007 \[SC-Low] periodattimestamp returns current period instead of historical

**Submitted on Nov 7th 2025 at 17:27:43 UTC by @Tomioka for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59007
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `periodAtTimestamp()` function is designed to return the period number for a given historical timestamp. However, due to the implementation of the private helper function `_sinceEpoch()`, it always uses `Time.timestamp()` (current block timestamp) instead of the provided timestamp parameter. This causes `periodAtTimestamp()` to always return the current period number regardless of what historical timestamp is passed to it.

### Vulnerability Details

**The Core Flaw:**

The `periodAtTimestamp()` function at line 246-250 calls `_sinceEpoch()` to calculate elapsed time:

```solidity
// Line 246-250: periodAtTimestamp function
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    // solhint-disable-next-line max-line-length
    return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}
```

The `_sinceEpoch()` helper function is defined at line 795-797:

```solidity
// Line 795-797: _sinceEpoch implementation
function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;
}
```

**Why This Is Wrong:**

The `periodAtTimestamp(uint48 timestamp)` function receives a `timestamp` parameter that should be used to calculate the period number at that point in time. However:

1. Line 247 correctly retrieves the `periodConfiguration` for the provided timestamp
2. Line 249 calls `_sinceEpoch(periodConfiguration.epoch)` which calculates `Time.timestamp() - epoch`
3. `Time.timestamp()` returns the **current** block timestamp, not the `timestamp` parameter passed to the function
4. The result is always calculated based on the current time, making the input parameter effectively ignored

**The Correct Implementation Should Be:**

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    // FIX: Use the provided timestamp parameter, not current time
    return periodConfiguration.startingPeriod + (timestamp - periodConfiguration.epoch) / periodConfiguration.duration;
}
```

**Root Cause Analysis:**

The bug stems from the design of `_sinceEpoch()` as a helper function that always uses the current timestamp. This pattern works correctly for functions like `currentPeriod()`, `currentPeriodStart()`, and `currentPeriodEnd()` which are meant to operate on the current time. However, it breaks the contract when used in `periodAtTimestamp()`, which needs to calculate periods for arbitrary historical (or future) timestamps.

**Functions Affected:**

The following functions use `_sinceEpoch()`:

* `periodAtTimestamp(uint48 timestamp)` : **BROKEN**, Should use provided timestamp
* `currentPeriodStart()` : Correct: Intended to use current time
* `currentPeriodEnd()` : Correct: Intended to use current time

### Impact Details

**Primary Impact: Historical Query Failure**

The primary consequence is that any attempt to query the period number for a historical timestamp will fail to return accurate historical data. Instead, it will always return the current period number.

**Secondary Impact: Dependent System Failures**

Any off chain systems, integrations, or internal logic that relies on `periodAtTimestamp()` to:

* Look up historical period numbers
* Validate timestamps against period boundaries
* Reconstruct historical state
* Perform period based calculations for past events

will receive incorrect data, leading to potential cascading failures.

**Tertiary Impact: User Confusion**

Users or integrators who call `periodAtTimestamp()` with a historical timestamp will receive confusing results:

* The function signature promises to return "the period number for the timestamp given"
* The return value will always be the current period
* No revert or error occurs, making the bug silent and hard to detect

**Practical Example:**

```solidity
// Period 0: Jan 1 - Jan 7  (epoch = Jan 1)
// Period 1: Jan 8 - Jan 14
// Period 2: Jan 15 - Jan 21 (current period)

uint48 jan5 = 1704412800; // January 5, 2024 timestamp

// Expected: periodAtTimestamp(jan5) should return 0
// Actual: periodAtTimestamp(jan5) returns 2 (current period)
```

**Affected Users:**

* Off chain systems querying historical period data
* Analytics tools and dashboards
* Third party integrations that rely on period lookups
* Smart contracts that call this function externally

### References

* Vulnerable code: `contracts/FirelightVault.sol:246-250`
* Helper function: `contracts/FirelightVault.sol:795-797`
* OpenZeppelin Time library: <https://docs.openzeppelin.com/contracts/5.x/api/utils#Time>
* NatSpec documentation at line 241-245 (describes intended behavior)

***

## Proof of Concept

## Proof of Concept

The complete runnable PoC : `test/poc_period_bug.js`.

### Execution

```bash

# Install dependencies
npm install

# Run the PoC test
npx hardhat test test/poc_period_bug.js
```

## Full PoC

```js

const { expect } = require('chai')
const { ethers } = require('hardhat')

const mineTime = async (seconds) => {
  await ethers.provider.send('evm_increaseTime', [seconds])
  await ethers.provider.send('evm_mine', [])
}

describe('PoC: periodAtTimestamp Bug (Low)', () => {
  async function deploySimpleVault() {
    const ERC20Factory = await ethers.getContractFactory('MinimalERC20')
    const token = await ERC20Factory.deploy('TestToken', 'TT', 6)
    await token.waitForDeployment()

    const VaultFactory = await ethers.getContractFactory('FirelightVault')
    const [deployer, limitUpdater, blocklister, pauser, periodUpdater] = await ethers.getSigners()
    const abiCoder = ethers.AbiCoder.defaultAbiCoder()
    const initParams = abiCoder.encode([
      'address','address','address','address','address','uint256','uint48'
    ],[
      deployer.address,
      limitUpdater.address,
      blocklister.address,
      pauser.address,
      periodUpdater.address,
      1_000_000_000n,
      7 * 24 * 60 * 60
    ])
    const vault = await ethers.upgrades.deployProxy(VaultFactory,[await token.getAddress(),'Share','Share',initParams])
    return { vault }
  }

  it('returns current period instead of historical for past timestamp', async () => {
    const { vault } = await deploySimpleVault()

    // Capture an early timestamp
    const startTs = (await ethers.provider.getBlock('latest')).timestamp

    // Advance time by almost two periods (default period 1 week)
    await mineTime(10 * 24 * 60 * 60) // 10 days

    const nowTs = (await ethers.provider.getBlock('latest')).timestamp
  const currentPeriodByView = await vault.currentPeriod()
  const supposedPastPeriod = await vault.periodAtTimestamp(startTs)

    // Expectation (correct logic): supposedPastPeriod should be lower than currentPeriodByView.
    // Actual behavior (bug): function uses current timestamp internally and returns currentPeriodByView.
    console.log('startTs:', startTs)
    console.log('nowTs:', nowTs)
    console.log('currentPeriod:', currentPeriodByView.toString())
    console.log('periodAtTimestamp(startTs):', supposedPastPeriod.toString())

    expect(supposedPastPeriod).to.equal(currentPeriodByView) // Showing the incorrect equality evidencing the bug.
  })
})

```

### Key Assertions

```javascript
// Capture early timestamp
const startTs = (await ethers.provider.getBlock('latest')).timestamp

// Advance time by 10 days
await mineTime(10 * 24 * 60 * 60)

// Get current state
const currentPeriodByView = await vault.currentPeriod()
const supposedPastPeriod = await vault.periodAtTimestamp(startTs)

// BUG: Historical query returns current period
expect(supposedPastPeriod).to.equal(currentPeriodByView)
```
