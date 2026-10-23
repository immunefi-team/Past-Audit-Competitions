# #59054 \[SC-Low] periodattimestamp returns incorrect period number

**Submitted on Nov 8th 2025 at 07:41:35 UTC by @dldLambda for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59054
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**

## Description

## Brief/Intro

The function periodAtTimestamp(uint48 timestamp) is intended to return the period number corresponding to a given timestamp. It uses \_sinceEpoch(periodConfiguration.epoch) to compute the offset.

\_sinceEpoch() always uses Time.timestamp() (the current timestamp) instead of the provided timestamp. As a result, no matter what historical timestamp is queried, the function calculates the period offset from the present, always returning the current period or a future period.

## Vulnerability Details

The periodAtTimestamp(uint48 timestamp) function promises to return the period number for any timestamp, but in reality it always returns current period:

```
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

```
    function _sinceEpoch(uint48 epoch) private view returns (uint48) {
        return Time.timestamp() - epoch;
    }
```

For example:

1. Vault deployed at epoch = 1,000,000, startingPeriod = 0
2. Period duration = 100 seconds
3. Historical timestamp: timestamp = 1,003,600
4. Correct period calculation: (timestamp - epoch) / duration = (1,003,600 - 1,000,000) / 100 = 36
5. Contract calculation: \_sinceEpoch(epoch) = Time.timestamp() - epoch = 1,010,000 - 1,000,000 = 10,000
6. Returned period: startingPeriod + \_sinceEpoch(epoch)/duration = 0 + 10,000 / 100 = 100

The same problem will occur if you pass a future timestamp.

However, at the moment, the function is only called in the currentPeriod function, which implies the current epoch. However, in the future, it may be used elsewhere, called in other places, and since it doesn't do what it says, this could cause major problems.

```
    /**
     * @notice Returns the current active period.
     * @return The current period number since contract deployment.
     */
    function currentPeriod() public view returns (uint256) {
        return periodAtTimestamp(Time.timestamp());
    }
```

## Impact Details

The function doesn't perform as expected, always returning the current period. If used elsewhere, any logic that relies on periodAtTimestamp() for historical accounting, rewards, or entitlement calculations will be broken.

## References

<https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L246>

## Proof of Concept

## Proof of Concept

Please add and run this test:

```
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers');
const { deployVault } = require('./setup/fixtures.js');
const { expect } = require('chai');
const { ethers } = require('hardhat');

describe('BUG: periodAtTimestamp() uses current time instead of parameter', function () {
  const DECIMALS = 6;
  const INITIAL_DEPOSIT_LIMIT = ethers.parseUnits('50000', DECIMALS);
  const PERIOD_DURATION = 604800; // 1 week in seconds

  let firelight_vault, deployment_time;

  before(async () => {
    const fixture = await loadFixture(
      deployVault.bind(null, {
        decimals: DECIMALS,
        initial_deposit_limit: INITIAL_DEPOSIT_LIMIT,
        period_configuration_duration: PERIOD_DURATION
      })
    );
    firelight_vault = fixture.firelight_vault;

    const currentPC = await firelight_vault.currentPeriodConfiguration();
    deployment_time = currentPC.epoch;

    console.log('\nTest Setup ');
    console.log('Deployment time (epoch):', deployment_time.toString());
    console.log('Period duration:', PERIOD_DURATION, 'seconds');
    console.log('Current period at deploy:', (await firelight_vault.currentPeriod()).toString());
  });

  it('demonstrates the bug clearly', async () => {
    console.log('\nBUG DEMONSTRATION');

    
    await time.increase(PERIOD_DURATION * 2);

    const currentPeriod = await firelight_vault.currentPeriod();
    console.log('Current period after time travel:', currentPeriod.toString());

    
    const historicalTimestamp = deployment_time + 300000n; 

    
    const contractPeriod = await firelight_vault.periodAtTimestamp(historicalTimestamp);

    // correct period
    const pc = await firelight_vault.currentPeriodConfiguration();
    const correctPeriod = pc.startingPeriod + (historicalTimestamp - pc.epoch) / pc.duration;

    console.log('Historical timestamp (should be period 0):', historicalTimestamp.toString());
    console.log('Contract returned periodAtTimestamp:', contractPeriod.toString());
    console.log('Current period:', currentPeriod.toString());
    console.log('Correct period (manually calculated):', correctPeriod.toString());

   
    expect(contractPeriod.toString(), 'BUG: periodAtTimestamp() returns current period instead of historical')
      .to.not.equal(correctPeriod.toString());
  });
});
```

You will see:

```
Test Setup 
Deployment time (epoch): 1762587557
Period duration: 604800 seconds
Current period at deploy: 0

BUG DEMONSTRATION
Current period after time travel: 2
Historical timestamp (should be period 0): 1762887557
Contract returned periodAtTimestamp: 2
Current period: 2
Correct period (manually calculated): 0
```
