# 59931 sc insight useless check

**Submitted on Nov 17th 2025 at 02:39:48 UTC by @Le\_Rems for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59931
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**

## Description

## Brief/Intro

The `periodConfigurationAtNumber()` function in `FirelightVault` contains a redundant validation check that will never execute. The function checks if `periodConfiguration.epoch == 0` and reverts, but this condition is impossible to reach given the function's loop logic and how period configurations are initialized. While this redundant check doesn't cause functional issues or fund loss, it wastes gas on every call.

## Vulnerability Details

The `FirelightVault` contract manages a dynamic array of `PeriodConfiguration` structures that define period boundaries over time. Each configuration contains an `epoch` , `duration` , and `startingPeriod` . These configurations are added during initialization and can be updated over time by users with the `PERIOD_CONFIGURATION_UPDATE_ROLE`.

The `periodConfigurationAtNumber()` function retrieves the appropriate period configuration for a given period number by iterating through the `periodConfigurations` array:

```solidity
function periodConfigurationAtNumber(uint256 periodNumber) external view returns (PeriodConfiguration memory) {
    uint256 length = periodConfigurations.length;
    if (length == 0) revert InvalidPeriod();

    PeriodConfiguration memory periodConfiguration;
    for (uint256 i = 0; i < length; i++) {
        if (periodNumber < periodConfigurations[i].startingPeriod)
            break;
        periodConfiguration = periodConfigurations[i];
    }
@>  if (periodConfiguration.epoch == 0) revert InvalidPeriod();
    return periodConfiguration;
}
```

The issue lies in the validation check. To understand why this check is unreachable, we must trace through the function's execution:

1. **Initial state**: `periodConfiguration` is declared as a memory struct. In Solidity, uninitialized structs have all fields set to zero values, so initially `periodConfiguration.epoch = 0`.
2. **First configuration guarantee**: The contract ensures `periodConfigurations.length >= 1` after initialization. During `initialize()`, the contract always calls `_addPeriodConfiguration(Time.timestamp(), initParams.periodConfigurationDuration)`, which adds the first configuration. This first configuration always has `startingPeriod = 0` as set in `_addPeriodConfiguration()`.
3. **Loop execution**: The loop starts at `i = 0`. For the first iteration:
   * The condition checks: `periodNumber < periodConfigurations[0].startingPeriod`
   * Since `periodConfigurations[0].startingPeriod = 0` and `periodNumber` is a `uint256` (always `>= 0`), the condition `periodNumber < 0` is always false
   * Therefore, the loop body executes: `periodConfiguration = periodConfigurations[0]`
   * This assignment sets `periodConfiguration.epoch` to `periodConfigurations[0].epoch`, which is `Time.timestamp()` from initialization (guaranteed to be non-zero as it's a real block timestamp)
4. **Epoch validation**: The epoch values are further protected by validation in `_addPeriodConfiguration()`:
   * For the first configuration: `if (newEpoch < Time.timestamp()) revert InvalidPeriodConfigurationEpoch();` ensures the epoch is at least the current timestamp (non-zero)
   * For subsequent configurations: `if (newEpoch < nextPeriodEnd() || ...)` ensures the epoch is always a valid future timestamp
5. **Unreachable check**: Since the loop is guaranteed to assign at least `periodConfigurations[0]` (which has a non-zero epoch), `periodConfiguration.epoch` can never be 0.

## Impact Details

The impact of this finding is primarily informational, affecting code quality rather than functionality:

**1. Gas Waste (Low Impact - Operational):**

* Every call to `periodConfigurationAtNumber()` performs an unnecessary `sload` operation to read `periodConfiguration.epoch` and a comparison check
* While minimal per call (\~100-200 gas), this accumulates over time and is wasteful

## References

* [`periodConfigurationAtNumber()` implementation](broken://pages/e56d99d6630a6b91b776a8c38701f01a5df5b8da#L237-L250)
* [`_addPeriodConfiguration()` function showing epoch validation](broken://pages/e56d99d6630a6b91b776a8c38701f01a5df5b8da#L825-L856)
* [`initialize()` function showing first configuration is always added](broken://pages/e56d99d6630a6b91b776a8c38701f01a5df5b8da#L178)
* [`periodConfigurationAtTimestamp()` showing identical redundant check pattern](broken://pages/e56d99d6630a6b91b776a8c38701f01a5df5b8da#L224)

## Proof of Concept

## Proof of Concept

```solidity

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {FirelightVault} from "../contracts/FirelightVault.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract FirelightVaultTest is Test {
    // Contracts
    FirelightVault public firelightVault;
    FirelightVault public firelightVaultImplementation;
    ERC1967Proxy public proxy;
    IERC20 public tokenContract;
    address public assetManager; // Mock asset manager for FAsset

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



    function test_uselessCheck() public {
        // the minimum of a uint is 0
        uint x = type(uint256).min;
        console.log("mibn uint256:", x);
        FirelightVault.PeriodConfiguration memory pc = firelightVault.currentPeriodConfiguration();
        console.log(pc.epoch);
        console.log(pc.duration);
        console.log(pc.startingPeriod);

        // calling the function with the lowest value possible
        firelightVault.periodConfigurationAtNumber(0);

        // in `if (periodNumber < periodConfigurations[i].startingPeriod)`
        // `periodNumber` is 0 and `periodConfigurations[i].startingPeriod` is 0
        // so 0 < 0 is false, so this line will always execute at least one time : `periodConfiguration = periodConfigurations[i];`
        // meaning `periodConfiguration.epoch` will always be diff than 0
        // meaning `if (periodConfiguration.epoch == 0) revert InvalidPeriod();` is dead logic and will never be executed
    }
}
```
