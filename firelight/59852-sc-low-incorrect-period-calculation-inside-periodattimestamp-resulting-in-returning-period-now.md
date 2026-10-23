# 59852 sc low incorrect period calculation inside periodattimestamp resulting in returning period now instead of period at given timestamp

**Submitted on Nov 16th 2025 at 12:34:49 UTC by @hcrlen for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59852
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `periodAtTimestamp(uint48 timestamp)` function is documented to return the period number for a given timestamp, but due to the `_sinceEpoch()` it incorrectly returns the current period regardless of the input timestamp. This breaks historical period queries and any off-chain systems relying on this function, but does not affect core vault operations or cause loss of funds.

## Vulnerability Details

The periodAtTimestamp() function accepts a timestamp parameter and promises to return the period number corresponding to that timestamp:

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
        return
            periodConfiguration.startingPeriod + 
@> _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
    }
```

inside \_sinceEpoch function it uses time.timeStamp()

```
function _sinceEpoch(uint48 epoch) private view returns (uint48) {
        return Time.timestamp() - epoch;
    }
```

The issue is in the \_sinceEpoch() function, which always uses Time.timestamp() (current block time) instead of the provided timestamp parameter.

## Impact Details

The function calculates the period based on the current time, not the requested historical timestamp. This means:

The same input (timestamp) returns different outputs depending on when the function is called

1. Unreliable Public Interface

* The function signature promises one behavior but delivers another
* Any integration depending on accurate historical period data will malfunction

2. Historical period queries are broken

* Any attempt to query historical period numbers returns incorrect results
* Off-chain systems cannot reliably verify past period data
* Analytics dashboards will show incorrect historical period information

3. The function violates its specification and natspec documentation

## References

FirelightVault.sol - periodAtTimestamp() - Line 246-250

FirelightVault.sol - \_sinceEpoch() - Line 795-297

## Proof of Concept

## Proof of Concept

This PoC uses the implementation contract directly without a proxy, as the bug exists in the core logic and is not related to proxy mechanics. The bug is reproducible regardless of deployment method. Using foundry Place the test file in test folder Run: `forge test --match-test test_periodAtTimestampReturnsIncorrectly -vv`

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import "lib/forge-std/src/Test.sol";
import "../../../contracts/FirelightVault.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract FirelightVaultPocTest is Test {
    FirelightVault public vault;
    IERC20 public token;

    function setUp() public {
        //deploy token
        token = IERC20(address(new MockERC20("Mock Token", "MTK", 18)));

        // Initialize vault with 1-day periods
        uint48 periodConfigurationDuration = 1 days;

        uint256 initialDepositLimit = 1000e18;
        // Create addresses
        address deployer = address(this);
        address rescuer = makeAddr("rescuer");
        address blocklister = makeAddr("blocklister");
        address pauser = makeAddr("pauser");
        address limitUpdater = makeAddr("limitUpdater");
        address periodConfigurationAddress = makeAddr("periodConfigurationUpdater");

        FirelightVault.InitParams memory initParams = FirelightVault.InitParams({
            defaultAdmin: deployer,
            limitUpdater: limitUpdater,
            blocklister: blocklister,
            pauser: pauser,
            periodConfigurationUpdater: periodConfigurationAddress,
            depositLimit: initialDepositLimit,
            periodConfigurationDuration: periodConfigurationDuration
        });

        bytes memory initParamsEncoded = abi.encode(initParams);

        vault = new FirelightVault();
        vault.initialize(token, "FLToken", "FLT", initParamsEncoded);
    }

    function test_periodAtTimestampReturnsIncorrectly() public {
        // Setup: Log initial configuration
        FirelightVaultStorage.PeriodConfiguration memory config = vault.currentPeriodConfiguration();
        console.log("Epoch:", config.epoch);
        console.log("Duration:", config.duration, "(1 day = 86400 seconds)");
        console.log("Starting Period:", config.startingPeriod);

        // Day 2: Calculate period at day 2 timestamp
        vm.warp(86400 * 2 + 1);
        uint48 day2Timestamp = uint48(block.timestamp);
        uint256 periodAtDay2 = vault.periodAtTimestamp(day2Timestamp);

        console.log("\n--- Day 2 ---");
        console.log("Timestamp:", day2Timestamp);
        console.log("Current period:", vault.currentPeriod());
        console.log("Period (calculated on day 2):", periodAtDay2);

        // Day 3: Query the SAME day 2 timestamp again
        vm.warp(86400 * 3 + 1);
        uint256 periodAtDay2_QueriedOnDay3 = vault.periodAtTimestamp(day2Timestamp);

        console.log("\n--- Day 3 ---");
        console.log("Current timestamp:", block.timestamp);
        console.log("Current period:", vault.currentPeriod());
        console.log("Period at day 2 (queried on day 3):", periodAtDay2_QueriedOnDay3);

        console.log("\n--- Bug Demonstration ---");
        console.log("Same input (day 2 timestamp):", day2Timestamp);
        console.log("Result on day 2:", periodAtDay2);
        console.log("Result on day 3:", periodAtDay2_QueriedOnDay3);
        console.log("Expected: Both should return", periodAtDay2);

        // This assertion FAILS, proving the bug
        assertEq(periodAtDay2_QueriedOnDay3, periodAtDay2, "Historical period query returned wrong result");
    }
}

// Simple mock ERC20 for testing
contract MockERC20 is IERC20 {
    string public name;
    string public symbol;
    uint8 public decimals;

    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
    }

    function totalSupply() external pure returns (uint256) {
        return 0;
    }

    function balanceOf(address) external pure returns (uint256) {
        return 0;
    }

    function transfer(address, uint256) external pure returns (bool) {
        return true;
    }

    function allowance(address, address) external pure returns (uint256) {
        return 0;
    }

    function approve(address, uint256) external pure returns (bool) {
        return true;
    }

    function transferFrom(address, address, uint256) external pure returns (bool) {
        return true;
    }
}

```

This should be the test result :

```
Ran 1 test for contracts/test/FirelightVaultPoc.t.sol:FirelightVaultPocTest
[FAIL: Historical period query returned wrong result: 3 != 2] test_periodAtTimestampReturnsIncorrectly() (gas: 52331)
Logs:
  Epoch: 1
  Duration: 86400 (1 day = 86400 seconds)
  Starting Period: 0
  
--- Day 2 ---
  Timestamp: 172801
  Current period: 2
  Period (calculated on day 2): 2
  
--- Day 3 ---
  Current timestamp: 259201
  Current period: 3
  Period at day 2 (queried on day 3): 3
  
--- Bug Demonstration ---
  Same input (day 2 timestamp): 172801
  Result on day 2: 2
  Result on day 3: 3
  Expected: Both should return 2
```
