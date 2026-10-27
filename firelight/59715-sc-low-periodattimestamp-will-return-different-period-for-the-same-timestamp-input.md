# #59715 \[SC-Low] periodattimestamp will return different period for the same timestamp input

**Submitted on Nov 15th 2025 at 03:54:46 UTC by @y4y for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59715
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In `FirelightVault::periodAtTimestamp`, the function returns the corresponding period at given timestamp. The view function should return the same period regardless how many other periods have passed, but in reality, periods passed can affect the returned value.

## Vulnerability Details

The issue lies in the usage of `_sinceEpoch`:

```solidity
    function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
        PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        // solhint-disable-next-line max-line-length
        return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
    }
```

And in `_sinceEpoch`, it uses the current timestamp instead of the supplied one:

```solidity
    function _sinceEpoch(uint48 epoch) private view returns (uint48) {
        return Time.timestamp() - epoch;
    }
```

Since in `periodAtTimestamp`, the period to query is at the given `timestamp` variable, not the current one. The discrepancy here would make function return incorrect period for the same timestamp after some time has passed.

## Impact Details

No other key accounting logic will be affected, only the view function `periodAtTimestamp` will return incorrect data.

## References

<https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L246>

## Proof of Concept

## Proof of Concept

Append the following test to `period_update.js`:

```javascript
  it('periodAtTimestamp returns different values for the same timestamp as time passes', async () => {
    const period_start = await firelight_vault.currentPeriodStart()
    const period_at_start_before = await firelight_vault.periodAtTimestamp(period_start)

    const duration = await current_period_duration()
    await time.increase(duration * 2)

    const period_at_start_after = await firelight_vault.periodAtTimestamp(period_start)

    expect(period_at_start_after).to.equal(period_at_start_before + 2n)
  })
```

The PoC will get the period at T from `periodAtTimestamp` at timestamp of T0, then fast forward to time T1. Normally, it's expected that the result period would be the same, but the `expect` statement will verify the period fetched at T1 will be 2 more than the one fetched at T0.
