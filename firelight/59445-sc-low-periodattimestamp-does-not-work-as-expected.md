# #59445 \[SC-Low] periodattimestamp does not work as expected

**Submitted on Nov 12th 2025 at 11:53:35 UTC by @zeroK for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59445
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

the function `periodAtTimestamp` meant to return specific period according to the timestamp that users set as input, for example users might require period number of timestamp of 10 jan then this function should return period 1(we assume the vault got active at 1 jan as timestamp which equal to period 0), however, the `periodAtTimestamp` does not return correct period for specific timestamp, this is because the `_sinceEpoch` calculates the time collapse according to current timestamp, which return the latest period(or current period), this function work as expected if we assume its invoked by `currentPeriod` only, but while the `periodAtTimestamp` is a public function, it can be invoked by anyone which lead to return of incorrect period value for users and third parties.

## Vulnerability Details

the function `periodAtTimestamp` implemented as shown below:

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
    } /


```

the comments above the function mentioned that it should return the period for the given timestamp, but this is not how its logic work because `_sinceEpoch` uses current timestamp to calculate collapsed time:

```solidity
    function _sinceEpoch(uint48 epoch) private view returns (uint48) {
        return Time.timestamp() - epoch;
    }

```

for our example, timestamp equal to jan 10 should lead to return period one, but due to this logic flow, the period returned is the latest period.

## Impact Details

the function `periodAtTimestamp` does not work as expected when it get invoked by users or third parties.

## References

<https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L246-L250>

<https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L795-L797>

## Proof of Concept

## Proof of Concept

add test below in rescue.ts and run `npx test hardhat` :

```ts

  describe("Period Calculation Bug POC", function () {
    it.only("POC: periodAtTimestamp returns wrong period for historical queries", async () => {
      const DECIMALS = 6;
      const DEPOSIT_LIMIT = ethers.parseUnits("100000", DECIMALS);

      // Deploy vault at CURRENT time (don't try to go backwards!)
      const { firelight_vault, config } = await loadFixture(
        deployVault.bind(null, { initial_deposit_limit: DEPOSIT_LIMIT })
      );

      // Record deployment time
      const deployTime = await time.latest();
      console.log("Vault deployed at timestamp:", deployTime);

      // Move forward 10 days
      await time.increase(10 * 24 * 60 * 60); // 10 days
      const time10Days = await time.latest();
      const periodAt10Days = await firelight_vault.currentPeriod();
      console.log("\n10 days later:");
      console.log("  Timestamp:", time10Days);
      console.log("  Current period:", periodAt10Days.toString());

      // Move forward another 20 days (30 days total)
      await time.increase(20 * 24 * 60 * 60); // 20 more days
      const time30Days = await time.latest();
      const currentPeriodNow = await firelight_vault.currentPeriod();
      console.log("\n30 days later (now):");
      console.log("  Timestamp:", time30Days);
      console.log("  Current period:", currentPeriodNow.toString());

      // Query what period it was 10 days after deployment
      const historicalQuery = await firelight_vault.periodAtTimestamp(
        time10Days
      );
      console.log("\nHistorical Query (10 days after deployment):");
      console.log("  periodAtTimestamp(10 days):", historicalQuery.toString());
      console.log("  Expected:", periodAt10Days.toString());
      console.log("  Actual:", historicalQuery.toString());

      if (historicalQuery.toString() === currentPeriodNow.toString()) {
        console.log("\n BUG CONFIRMED!");
        console.log(
          "Historical query returned CURRENT period instead of historical period!"
        );
        console.log(
          "This proves the function uses Time.timestamp() instead of the parameter!"
        );
      } else if (historicalQuery.toString() === periodAt10Days.toString()) {
        console.log("\n NO BUG - Function works correctly!");
      }

      // Show the discrepancy
      console.log("\nDiscrepancy:");
      console.log("  Query was for:", time10Days, "(10 days after deploy)");
      console.log("  Should return period:", periodAt10Days.toString());
      console.log("  Actually returns period:", historicalQuery.toString());
      console.log("  Current period is:", currentPeriodNow.toString());
    });
  });



```
