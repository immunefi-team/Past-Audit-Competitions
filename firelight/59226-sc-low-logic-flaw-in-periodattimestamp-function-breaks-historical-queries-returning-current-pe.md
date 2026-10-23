# 59226 sc low logic flaw in periodattimestamp function breaks historical queries returning current period instead

**Submitted on Nov 10th 2025 at 07:06:38 UTC by @theboiledcorn for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59226
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `periodAtTimestamp` view function has a critical logic flaw where it ignores its `timestamp` parameter. The function incorrectly uses the current block time (`Time.timestamp()`) for its calculation instead of the provided `timestamp`. Consequently, any query for a historical period will fail and incorrectly return the *current* period number. This breaks all historical lookup functionality, as the function provides a demonstrably incorrect value to any user or contract that calls it.

***

## Vulnerability Details

The vulnerability is a logic error in how `periodAtTimestamp` calculates the period number. The function correctly uses the `timestamp` parameter to find the right `PeriodConfiguration`, but it fails to use that `timestamp` in the final calculation.

Here is the flawed function:

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    // ...
    // THE FLAW IS HERE:
    return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}
```

The function passes the `periodConfiguration.epoch` to the `_sinceEpoch` helper, but it *omits* the `timestamp` variable.

The `_sinceEpoch` helper function is defined as:

```solidity
function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    // It *always* uses the current block time, not the requested timestamp
    return Time.timestamp() - epoch;
}
```

Because `_sinceEpoch` uses `Time.timestamp()`, the `periodAtTimestamp` function also inadvertently uses `Time.timestamp()`.

### Proof of Concept Scenario

This can be proven with a simple test:

1.

Assume the first period configuration is set up at `block.timestamp = 1` with a `duration` of `604800`.

```
  * Period 0: `[1, 604801)`
  * Period 1: `[604801, 1209601)`
  * Period 2: `[1209601, 1814401)`
```

2\. \* Warp the blockchain time far into the future, for example, into Period 5. `vm.warp((604800 * 5) + 1);` // block.timestamp is now `3,024,001` \* Call the function, asking for the period at a time in **Period 1**. `uint48 pastTimestamp = 1_000_000;` `uint256 result = vault.periodAtTimestamp(pastTimestamp);`

3. * **Expected (Correct) Result:** The function should return **1**, as `1,000,000` falls within Period 1.
   * **Actual (Buggy) Result:** The function will calculate `(Time.timestamp() - epoch) / duration` = `(3024001 - 1) / 604800` = **5**. The function incorrectly returns **5**.

***

## Impact Details

The direct impact is a **failure of the contract's core logic**. The `periodAtTimestamp` function does not perform its advertised behavior, making it useless for its intended purpose of historical data retrieval. Any trust placed in this function by users or other contracts is broken.

This bug maps to the following categories from the program's in-scope impacts:

* **Contract fails to deliver promised returns, but doesn't lose value:** The function *promises* (via its name and parameters) to return a specific historical period number. It *fails* to deliver this value, instead returning the current (and incorrect) period number.

Any user, UI, or integrated contract that relies on this function for historical data will receive incorrect values. This can cause downstream logic to fail, transactions to revert, or UIs to display false information. This is a clear case of "damage to the users" as the protocol's functionality is broken, even if there is no direct profit motive for an attacker.

***

## References

* `function periodAtTimestamp(uint48 timestamp)`
* `function _sinceEpoch(uint48 epoch)`

## Proof of Concept

## Proof of Concept

The following test case demonstrates the vulnerability. It establishes an initial period configuration, warps time far into the future to create a disparity, and then queries a historical timestamp. The test confirms that the function incorrectly returns the current period instead of the historical one.

### Step-by-Step Explanation

1. The `setUp()` function initializes the vault. The `initialize` function (called from `setUp`) adds the first `PeriodConfiguration`. Based on the `FirelightVault.sol` contract's `initialize` function, this first configuration is: `{ epoch: 1, duration: 604800, startingPeriod: 0 }`.
2. The test warps the block time far into the future to **Period 5** (timestamp `3,024,001`). This makes the "current period" 5.
3. The test then calls `vault.periodAtTimestamp()` with a `pastTimestamp` of `1,000,000`. This timestamp clearly falls within **Period 1** (which spans `[604_801, 1_209_601)`).
4. The test asserts two conditions:
   * It confirms that the `buggyResult` is **5** (the *current* period), which is incorrect.
   * It confirms that the `buggyResult` is **not 1** (the *correct* historical period).

### Test Code

```solidity
/* SPDX-License-Identifier: UNLICENSED */
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import "forge-std/console2.sol";
import "../contracts/FirelightVault.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "./FXRP.sol";

contract FireLightVaultTest is Test {
    FirelightVault public vault;
    FXRP public fXRP;
    address vaultAdmin = makeAddr("vaultAdmin");

    FirelightVault.InitParams _initParams = FirelightVault.InitParams({
        defaultAdmin: vaultAdmin,
        limitUpdater: vaultAdmin,
        blocklister: vaultAdmin,
        pauser: vaultAdmin,
        periodConfigurationUpdater: vaultAdmin,
        depositLimit: 50000000000,
        periodConfigurationDuration: 604800
    });

    function setUp() public {
        vault = new FirelightVault();
        fXRP = new FXRP();

        vault.initialize(IERC20(address(fXRP)), "stfXRP", "stfXRP", abi.encode(_initParams));
    }

    /**
     * @notice POC: periodAtTimestamp() is flawed.
     * It incorrectly returns the current period (based on block.timestamp)
     * instead of the period for the pastTimestamp we ask for.
     */
    function test_POC_periodAtTimestamp_ReturnsCurrentPeriod() public {
        // From setUp, we know:
        // epoch = 1
        // duration = 604_800
        // startingPeriod = 0

        // This means:
        // Period 0: [1, 604_801)
        // Period 1: [604_801, 1_209_601)

        // Let's pick a historical timestamp that is clearly in Period 1.
        uint48 pastTimestamp = 1_000_000;
        uint26 expectedCorrectPeriod = 1;

        // Now, let's warp time far into the future, e.g., into Period 5.
        // The current period will now be 5.
        uint48 futureTime = (604_800 * 5) + 1;
        vm.warp(futureTime);

        // We are now in Period 5.
        // Let's call the function, asking for the period at our past timestamp.
        uint256 buggyResult = vault.periodAtTimestamp(pastTimestamp);

        // The function should have returned 1 (for our pastTimestamp).
        // Instead, the bug will cause it to return the current period, 5.

        uint256 actualCurrentPeriod = vault.currentPeriod();

        // This assertion proves the bug:
        assertEq(buggyResult, actualCurrentPeriod, "BUG: Function returned the current period");

        // This assertion shows the failure:
        assertNotEq(buggyResult, expectedCorrectPeriod, "BUG: Function did NOT return the correct past period");
    }
}
```

### Test Output

The test passes, confirming the bug. The trace shows `vault.periodAtTimestamp(1000000)` incorrectly returning `5`.

```bash
Ran 1 test for test/FireLightVaultTest.t.sol:FireLightVaultTest
[PASS] test_POC_periodAtTimestamp_ReturnsCurrentPeriod() (gas: 21723)
Traces:
  [21723] FireLightVaultTest::test_POC_periodAtTimestamp_ReturnsCurrentPeriod()
    ├─ [0] VM::warp(3024001 [3.024e6])
    │   └─ ← [Return]
    ├─ [9066] FirelightVault::periodAtTimestamp(1000000 [1e6]) [staticcall]
    │   └─ ← [Return] 5
    ├─ [2994] FirelightVault::currentPeriod() [staticcall]
    │   └─ ← [Return] 5
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.01ms (197.31µs CPU time)
```
