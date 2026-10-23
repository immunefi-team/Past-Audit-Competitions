# #59605 \[SC-Low] logic error in periodattimestamp returns incorrect periods

**Submitted on Nov 14th 2025 at 00:37:09 UTC by @zcai for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59605
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `periodAtTimestamp()` function ignores its input timestamp parameter and instead uses the current block time for period calculations. This causes incorrect period results for any non-current timestamp and potential underflow reverts when querying future timestamps under scheduled future configurations. The vulnerability compromises data integrity and may cause denial-of-service for certain timestamp queries.

NOTE: This bug in `periodAtTimestamp()` erroneously uses block.timestamp instead of the passed in timestamp. However, the critical `currentPeriod()` function which relies on this function luckily passes in the block.timestamp as the argument so it is not affected. However, integrators relying on the return value of `periodAtTimestamp()` are directly affected, hence the severity is reduced to medium.

## Vulnerability Details

The `periodAtTimestamp(uint48 timestamp)` function is designed to compute the period number for an arbitrary timestamp, but contains a critical implementation flaw. Instead of using the provided timestamp parameter in its calculations, the function relies on `_sinceEpoch(periodConfiguration.epoch)`, which internally uses `Time.timestamp()` (the current block time) rather than the input timestamp.

The function currently computes:

```
periodConfiguration.startingPeriod + (Time.timestamp() - periodConfiguration.epoch) / periodConfiguration.duration
```

When it should compute:

```
periodConfiguration.startingPeriod + (timestamp - periodConfiguration.epoch) / periodConfiguration.duration
```

This implementation error manifests in two critical ways. First, the function returns the period for the current time under the configuration selected for the provided timestamp, not the period for the provided timestamp itself. Second, when a future timestamp falls under a scheduled future configuration with an epoch greater than the current time, the calculation `Time.timestamp() - epoch` results in an unsigned integer underflow in Solidity 0.8, causing the transaction to revert.

## Impact Details

The function compromises data integrity by returning incorrect period numbers for any timestamp that differs from the current block time. When future timestamps fall under scheduled future configurations, the function reverts due to underflow, creating a denial-of-service condition for such queries. While core protocol flows using `currentPeriod()` remain functional since they pass the current timestamp, any integrations or extensions relying on `periodAtTimestamp()` for historical or future period mapping will behave incorrectly.

## References

[contracts/FirelightVault.sol:246-250](https://github.com/firelight-protocol/firelight-core/blob/b26d47eebd0bf965c2979c8bf779d334a1c7bab6/contracts/FirelightVault.sol#L246-L250)

[contracts/FirelightVault.sol:795-797](https://github.com/firelight-protocol/firelight-core/blob/b26d47eebd0bf965c2979c8bf779d334a1c7bab6/contracts/FirelightVault.sol#L795-L797)

## Link to Proof of Concept

<https://gist.github.com/i-am-zcai/9c0fb1b144d706e764522d95c5cd7cf9>

## Proof of Concept

## Proof of Concept

const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers') const { deployVault } = require('../setup/fixtures.js') const { expect } = require('chai')

/\*\*

* PoC: periodAtTimestamp ignores its input timestamp parameter
*
* Demonstrates two bugs in periodAtTimestamp():
* 1. Returns wrong period values because it uses Time.timestamp() instead of the input
* 2. Reverts on future timestamps under scheduled future configs due to underflow \*/ describe('PoC - periodAtTimestamp ignores input timestamp', function () { const DECIMALS = 6 const DEPOSIT\_LIMIT = 100000n \* 10n \*\* BigInt(DECIMALS) const PERIOD\_DURATION = 172800 // 2 days

let firelight\_vault, period\_configuration\_updater

before(async () => { ({ firelight\_vault, period\_configuration\_updater } = await loadFixture( deployVault.bind(null, { initial\_deposit\_limit: DEPOSIT\_LIMIT, period\_configuration\_duration: PERIOD\_DURATION }) )) })

it('demonstrates periodAtTimestamp returns wrong period for past timestamps', async () => { const currentPeriod = await firelight\_vault.currentPeriod() const currentTime = await time.latest()

```
// Query past time - should return previous period but returns current
const pastTime = currentTime - PERIOD_DURATION
const periodAtPast = await firelight_vault.periodAtTimestamp(pastTime)

expect(periodAtPast).to.equal(currentPeriod) // ← Bug: should be currentPeriod - 1
console.log(`\n  Bug #1: Wrong period for past timestamp`)
console.log(`    periodAtTimestamp(now - 1 period): ${periodAtPast}`)
console.log(`    Expected: ${currentPeriod - 1n}, Got: ${periodAtPast}`)
```

})

it('demonstrates periodAtTimestamp reverts on future timestamp with future config', async () => { // Schedule a future period configuration const currentPeriodEnd = await firelight\_vault.currentPeriodEnd() const futureEpoch = Number(currentPeriodEnd) + PERIOD\_DURATION const futureDuration = 604800 // 1 week

```
await firelight_vault
  .connect(period_configuration_updater)
  .addPeriodConfiguration(futureEpoch, futureDuration)

// Query a timestamp that falls under the future configuration
// This will cause _sinceEpoch to try: Time.timestamp() - futureEpoch
// Since futureEpoch > Time.timestamp(), this underflows and reverts
const futureTimestamp = futureEpoch + 1000

await expect(
  firelight_vault.periodAtTimestamp(futureTimestamp)
).to.be.reverted // Underflow in _sinceEpoch

console.log(`\n  Bug #2: Reverts on future timestamp with future config`)
console.log(`    Future config epoch: ${futureEpoch}`)
console.log(`    Query timestamp: ${futureTimestamp}`)
console.log(`    Result: REVERTS due to underflow in _sinceEpoch()`)
```

}) })
