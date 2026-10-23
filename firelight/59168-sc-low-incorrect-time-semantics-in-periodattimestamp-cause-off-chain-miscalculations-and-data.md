# 59168 sc low incorrect time semantics in periodattimestamp cause off chain miscalculations and data inconsistency

**Submitted on Nov 9th 2025 at 14:33:19 UTC by @Arkindyo for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59168
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `periodAtTimestamp(uint48 timestamp)` function in `FirelightVault.sol` returns the current period number based on the current block timestamp, not the period for the provided `timestamp`.

\##Vulnerability Details

The `periodAtTimestamp` function is documented to return “the period number for the given timestamp”, but it computes with `Time.timestamp()` (current block time) instead of the provided `timestamp`. Although it correctly selects the period configuration applicable to the provided `timestamp` via `periodConfigurationAtTimestamp(timestamp)`, the quotient uses the elapsed time since `epoch` based on “now” rather than the provided `timestamp`.

[firelight-core/contracts/FirelightVault.sol::periodAtTimestamp#249](https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L249)

[firelight-core/contracts/FirelightVault.sol::\_sinceEpoch#796](https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L796)

```solidity
  function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
        PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        // solhint-disable-next-line max-line-length
        return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}

  function _sinceEpoch(uint48 epoch) private view returns (uint48) {
      return Time.timestamp() - epoch;
  }

```

## Impact Details

* Incorrect historical/future queries: Any off-chain analytics, dashboards, or integrators using `periodAtTimestamp(pastOrFutureTs)` will receive the period number for “now” (under the configuration that would apply to `timestamp`), not for the actual `timestamp`.
* Mis-scheduling and mislabeling: Systems that plan actions or label data by period number may misassign entries, leading to misleading reports and operational errors (e.g., wrong “batch” attribution or period-based KPIs).
* User confusion and governance optics: Period-based histories and charts may appear inconsistent, undermining trust and complicating audits.

## Recommended Fix

Update the computation to use the provided `timestamp`:

```solidity
  function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
       PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        // solhint-disable-next-line max-line-length
-       return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
+       return periodConfiguration.startingPeriod + (timestamp - periodConfiguration.epoch) / periodConfiguration.duration;
  }
```

* The contract enforces left-closed, right-open intervals for period divisions. With the correct formula, when `timestamp == pc.epoch + k * pc.duration`, the function should return `pc.startingPeriod + k`.
* Period alignment and configuration transitions are already guarded by `_addPeriodConfiguration` and `periodConfigurationAtTimestamp`, so using `(timestamp - pc.epoch)` is safe (no underflow) because `timestamp >= pc.epoch` for the selected configuration.

## References

Code: `firelight-core/contracts/FirelightVault.sol`, `periodAtTimestamp(uint48 timestamp)` and `_sinceEpoch(uint48 epoch)`.

[firelight-core/contracts/FirelightVault.sol::periodAtTimestamp#249](https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L249)

[firelight-core/contracts/FirelightVault.sol::\_sinceEpoch#796](https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L796)

## Proof of Concept

## Proof of Concept

The `periodAtTimestamp` function uses the current block time instead of the input timestamp, causing incorrect period numbers when querying past or future timestamps. This test `test_PeriodAtTimestamp_UsesNowInsteadOfArg()` publicconstructs a period configuration that has already started in the past, then queries with a "past timestamp" to assert that the returned value equals the period based on the `current` time rather than the correct one based on the query timestamp.

```bash
Ran 1 test for contracts/test/FirelightVaultUpgradeTest.sol:FirelightVaultUpgradeTest
[PASS] test_PeriodAtTimestamp_UsesNowInsteadOfArg() (gas: 54414)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.17ms (201.97µs CPU time)
```

```solidity

/* SPDX-License-Identifier: UNLICENSED */

pragma solidity 0.8.28;

import {FirelightVault} from "../FirelightVault.sol";
import {Checkpoints} from "../lib/Checkpoints.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";

contract FirelightVaultUpgradeTest is FirelightVault, Test {
    using Checkpoints for Checkpoints.Trace256;
    using SafeERC20 for IERC20;

    function updateVersion(uint256 version) public {
        contractVersion = version;
    }

    // POC: test_PeriodAtTimestamp_UsesNowInsteadOfArg
    function test_PeriodAtTimestamp_UsesNowInsteadOfArg() public {
        uint48 epoch0 = uint48(block.timestamp);
        PeriodConfiguration memory pc = PeriodConfiguration({
            epoch: epoch0,
            duration: uint48(100),
            startingPeriod: 0
        });
        periodConfigurations.push(pc);

        uint48 queryTs = epoch0 + 1;
        uint256 expectedCorrect = pc.startingPeriod +
            (uint256(queryTs) - uint256(pc.epoch)) /
            uint256(pc.duration);

        vm.warp(uint256(epoch0) + 150);
        uint256 nowBased = pc.startingPeriod +
            (uint256(uint48(block.timestamp)) - uint256(pc.epoch)) /
            uint256(pc.duration);

        uint256 actual = periodAtTimestamp(queryTs);

        require(
            actual == nowBased,
            "periodAtTimestamp returns now-based period instead of ts-based"
        );
        require(
            actual != expectedCorrect,
            "periodAtTimestamp should not equal correct period for past ts"
        );
    }
}

```
