# #59928 \[SC-Low] incorrect period calculation periodattimestamp function

**Submitted on Nov 17th 2025 at 02:26:03 UTC by @Le\_Rems for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59928
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `periodAtTimestamp()` function in `FirelightVault` is designed to return the period number corresponding to any given timestamp. However, the implementation incorrectly uses the current block timestamp instead of the provided parameter when calculating the period number. This causes the function to return incorrect results for any timestamp that differs from the current time.

## Vulnerability Details

The `FirelightVault` contract implements a period-based system where time is divided into discrete periods, each defined by a `PeriodConfiguration` containing an epoch timestamp, duration, and starting period number.

The `periodAtTimestamp()` function is a public view function intended to determine which period number corresponds to a specific timestamp. This function should enable querying periods for any point in time—past, present, or future—which is essential for historical queries, period validation, and integration with external systems.

The function implementation is as follows:

[periodAtTimestamp()](https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol?utm_source=immunefi#L246C1-L250C6)

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}
```

The function first correctly retrieves the appropriate `PeriodConfiguration` for the given timestamp by calling `periodConfigurationAtTimestamp(timestamp)`. However, when calculating the actual period number, it calls `_sinceEpoch(periodConfiguration.epoch)`, which is defined as:

[\_sinceEpoch()](https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol?utm_source=immunefi#L795C4-L797C6)

```solidity
function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;
}
```

The root cause is that `_sinceEpoch()` always uses `Time.timestamp()` (the current block timestamp) rather than accepting and using the `timestamp` parameter provided to `periodAtTimestamp()`. This means:

1. When querying the current period, the function works correctly because `timestamp` equals `Time.timestamp()`
2. When querying a past timestamp, the function retrieves the correct configuration but calculates the period number using the current time, returning a period that would be correct now but incorrect for the queried time
3. When querying a future timestamp, the calculation is similarly incorrect

For example, consider a scenario where:

* The vault has been operational for 100 periods, each lasting 30 days
* A period configuration has an epoch at timestamp T0 and duration of 30 days
* At timestamp T0 + 15 days, the period number should be `startingPeriod + 0` (still in the first period)
* At timestamp T0 + 45 days (15 days into the future from the current time T0 + 30 days), if someone queries `periodAtTimestamp(T0 + 15)`, the function would:
  1. Correctly identify the period configuration for T0 + 15 ✓
  2. But calculate using `(T0 + 30) - epoch` instead of `(T0 + 15) - epoch`, returning `startingPeriod + 1` instead of `startingPeriod + 0` ✗

The issue affects the contract's API contract: the function signature and documentation promise to return the period for "the timestamp given," but the implementation violates this promise for any timestamp other than the current time.

## Impact Details

The impact manifests in several ways:

**1. The function doesn't behave as expected**

* Any external system or smart contract that queries `periodAtTimestamp()` with historical timestamps will receive incorrect period numbers

**2. Future Functionality Risks**

* If the contract is upgraded or extended to validate withdrawals, deposits, or claims based on historical periods (e.g., validating that a withdrawal was requested in a specific past period), the bug would cause incorrect validations

## Proof of Concept

// SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.28;

import {Test, console} from "forge-std/Test.sol"; import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol"; import {FirelightVault} from "../contracts/FirelightVault.sol"; import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract FirelightVaultTest is Test { // Contracts FirelightVault public firelightVault; FirelightVault public firelightVaultImplementation; ERC1967Proxy public proxy; IERC20 public tokenContract; address public assetManager; // Mock asset manager for FAsset

```
// Roles and addresses
address public deployer;
address public rescuer;
address public blocklister;
address public pauser;
address public limitUpdater;
address public periodConfigurationUpdater;
address public user1;
address public user2;
address public user3;

// Configuration
uint8 public constant DECIMALS = 6;
uint256 public constant INITIAL_DEPOSIT_LIMIT = 20000 * 10**DECIMALS; // 20k tokens
uint256 public constant DEPOSIT_AMOUNT = 10000 * 10**DECIMALS; // 10k tokens
uint48 public constant PERIOD_CONFIGURATION_DURATION = 172800; // 2 days
uint48 public constant PERIOD_TARGET_DURATION = 604800; // 1 week

// Default config
string public constant VAULT_NAME = "stfXRP";
string public constant VAULT_SYMBOL = "stfXRP";
string public constant UNDERLYING_NAME = "fXRP";

    struct PeriodConfiguration {
    uint48 epoch;
    uint48 duration;
    uint256 startingPeriod; // @audit might able to opti this to fit in asingle u256 and save gas, especially if `startingPeriod` is only a counter
}

// Events (if needed for testing)
event DepositLimitUpdated(uint256 limit);
event PeriodConfigurationAdded(FirelightVault.PeriodConfiguration periodConfiguration);

function setUp() public {

    vm.warp(51651315616513);
    // Setup addresses
    deployer = address(this); // Test contract acts as deployer
    rescuer = address(0x1);
    blocklister = address(0x2);
    pauser = address(0x3);
    limitUpdater = address(0x4);
    periodConfigurationUpdater = address(0x5);
    user1 = address(0x10);
    user2 = address(0x11);
    user3 = address(0x12);
    assetManager = address(0x20);

    // Deploy mock token contract (FAsset) or use actual deployment
    // For now, using a simple mock approach - you may need to deploy actual FAsset
    tokenContract = IERC20(address(0x15)); //deployMockToken();
    
    // Deploy FirelightVault implementation
    //firelightVaultImplementation = new FirelightVault();

    // Encode InitParams
    FirelightVault.InitParams memory initParams = FirelightVault.InitParams({
        defaultAdmin: deployer,
        limitUpdater: limitUpdater,
        blocklister: blocklister,
        pauser: pauser,
        periodConfigurationUpdater: periodConfigurationUpdater,
        depositLimit: INITIAL_DEPOSIT_LIMIT,
        periodConfigurationDuration: PERIOD_CONFIGURATION_DURATION
    });

    // Encode init params for initialize function
    bytes memory initParamsEncoded = abi.encode(
        initParams.defaultAdmin,
        initParams.limitUpdater,
        initParams.blocklister,
        initParams.pauser,
        initParams.periodConfigurationUpdater,
        initParams.depositLimit,
        initParams.periodConfigurationDuration
    );

    // Get vault instance
    firelightVault = new FirelightVault();
    firelightVault.initialize(tokenContract, VAULT_NAME, VAULT_SYMBOL, initParamsEncoded);

    console.log("tokenContract", address(firelightVault));
}

///////////////////////////////////////////////////////////////////////////



function test_brokenFunction() public view {
    uint48 currentTimestamp = uint48(block.timestamp);
    uint48 futureTimestamp = currentTimestamp + PERIOD_CONFIGURATION_DURATION * 5;

    uint256 currentPeriod = firelightVault.periodAtTimestamp(currentTimestamp);
    uint256 futurePeriod = firelightVault.periodAtTimestamp(futureTimestamp);

    // the future period should be currentPeriod + 5 and not equal
    // note: this bug is effective also on timestamp in the past
    assertEq(
        currentPeriod,
        futurePeriod,
        "periodAtTimestamp should change when the queried timestamp moves forward"
    );
}
```

}
