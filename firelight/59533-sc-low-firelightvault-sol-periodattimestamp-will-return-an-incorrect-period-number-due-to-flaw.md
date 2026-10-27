# 59533 sc low firelightvault sol periodattimestamp will return an incorrect period number due to flawed logic

**Submitted on Nov 13th 2025 at 07:33:03 UTC by @maggie for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59533
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

FirelightVault.sol#periodAtTimestamp() is used to get the period number for the given timestamp. In order to get the period number, we should get the period configuration for the give timestamp first, then add passed period number to startingPeriod of period configuration. However, the passed period number is calculated with current timestamp instead of the give timestamp, which will cause incorrect period number.

## Vulnerability Details

```
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    // solhint-disable-next-line max-line-length
    return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}

function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;
}
```

periodConfiguration for the give timestamp is fetched, then add periodConfiguration.startingPeriod with passed period number. Passed period number = (time since epoch) /periodConfiguration.duration :

```
(timestamp - periodConfiguration.epoch) / periodConfiguration.duration.
```

In function periodAtTimestamp(uint48 timestamp), passed period number is calculated as following:

```
_sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration  
```

which is :

```
(Time.timestamp() - periodConfiguration.epoch) / periodConfiguration.duration.
```

If the give timestamp is not the current time, this function could return an incorrect period number.

We assume the following period configuration for the give time, 2025-11-03 10:00:00.

1, epoch: 2025-11-01 08:00,

2, duration: 1 days,

3, startingPeriod: 10

Now, user want to get period number for 2025-11-03 10:00:00, the expected period number should be:

startingPeriod + passed period number =

10 + (2025-11-03 10:00:00 - 2025-11-01 08:00) / 1days =

10 + 2 = 12

In fact, if users query the timestamp(2025-11-03 10:00:00) at 2025-11-06 10:00:00, user will get 10 + (2025-11-06 10:00:00 - 2025-11-01 08:00) / 1days = 10 + 5 = 15.

If users query the timestamp(2025-11-03 10:00:00) at 2025-11-07 10:00:00, user will get 10 + (2025-11-07 10:00:00 - 2025-11-01 08:00) / 1days = 10 + 6 = 16.

## Impact Details

FirelightVault.sol#periodAtTimestamp() is a public function opened to all users, if the given timestamp is not the current timestamp, users could get an incorrect period number, which will affect following operations.

## References

<https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol#L249>

## Proof of Concept

## Proof of Concept

PoC steps:

1, add new period configurations.

2, set current time to new\_epoch+7days, and query period number for (new\_epoch + 2days)

3, set current time to new\_epoch + 14days, and query period number for (new\_epoch + 2days)

4, period number in step 2 and step 3 should be the same.

Please add the following code to period\_update.js and run the test:

```
 it('test periodAtTimestamp', async () => {
 const current_period_end = await firelight_vault.currentPeriodEnd(),
      new_epoch = Number(current_period_end) + await current_period_duration()

await firelight_vault.connect(period_configuration_updater).addPeriodConfiguration(new_epoch, PERIOD_TARGET_DURATION)

const [period_epoch, period_duration, period_start] = await firelight_vault.periodConfigurations(1),
      current_period = await firelight_vault.currentPeriod()

expect(period_epoch).to.equal(new_epoch)
expect(period_duration).to.equal(PERIOD_TARGET_DURATION)
expect(period_start).to.equal(current_period + 2n)

await time.increaseTo(new_epoch + PERIOD_TARGET_DURATION);  //new epoch + 7 days
const period_number1 = await firelight_vault.periodAtTimestamp(new_epoch + 86400*2);
await time.increaseTo(new_epoch + PERIOD_TARGET_DURATION*2);  //new epoch + 14 days
const period_number2 = await firelight_vault.periodAtTimestamp(new_epoch + 86400*2);
await expect(period_number1).to.equal(period_number2)
 })
```
