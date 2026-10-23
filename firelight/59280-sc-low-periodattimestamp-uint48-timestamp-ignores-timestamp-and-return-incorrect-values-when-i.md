# 59280 sc low periodattimestamp uint48 timestamp ignores timestamp and return incorrect values when it is not time timestamp&#x20;

**Submitted on Nov 10th 2025 at 16:21:08 UTC by @axolot for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59280
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`periodAtTimestamp(timestamp)` computes the period using the current time instead of the `timestamp` argument. Internal logic for deposits/withdrawals is not affected, but the viewer is broken as external callers will get incorrect historical or future period indice.

## Vulnerability Details

This is the function. The issue is that it uses `_sinceEpoch` which does not take into account `timestamp`

```solidity
    function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
        PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        // solhint-disable-next-line max-line-length
        return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
    }
```

```solidity
    function _sinceEpoch(uint48 epoch) private view returns (uint48) {
        return Time.timestamp() - epoch;//@audit - always using current timestamp
    }
```

So `periodAtTimestamp(T)` will always equal the current active period `currentPeriod()`, not the period corresponding to timestamp `T`.

This is what the function should be

```diff
    function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
        PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        // solhint-disable-next-line max-line-length
-        return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
+        return periodConfiguration.startingPeriod + (timestamp - periodConfiguration.epoch) / periodConfiguration.duration;
    }
```

## Impact Details

Broken viewer for `timestamp != Time.timestamp()`. This falls into the `Contract fails to deliver promised returns, but doesn't lose value` category

## Proof of Concept

## Proof of Concept

1- set up Foundry in the repo

```bash
forge init --force
```

2- Install the OpenZeppelin libraries

```bash
forge install OpenZeppelin/openzeppelin-contracts-upgradeable --no-commit
forge install OpenZeppelin/openzeppelin-contracts --no-commit
```

3- Create the remapping file `remappings.txt` in the root directory and paste this

```
@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts/
@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
erc4626-tests/=lib/openzeppelin-contracts-upgradeable/lib/erc4626-tests/
forge-std/=lib/forge-std/src/
halmos-cheatcodes/=lib/openzeppelin-contracts-upgradeable/lib/halmos-cheatcodes/src/
openzeppelin-contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/
openzeppelin-contracts/=lib/openzeppelin-contracts/
```

4- Paste the following file in `test/FirelightVault.t.sol` (you can temporarily rename the hardhat `test` folder in `test-hardhat` to separate them)

Run with `forge test`. It shows how the function does not return the correct value when using an argument different than the current timestamp.

```solidity

// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {FirelightVault} from "../src/FirelightVault.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockERC20 is ERC20 {
    constructor() ERC20("Mock Token", "MOCK") {
        _mint(msg.sender, 1000000 * 10**18);
    }
    
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract FirelightVaultTest is Test {
    FirelightVault public vault;
    MockERC20 public asset;
    
    address public admin = address(1);
    address public limitUpdater = address(2);
    address public blocklister = address(3);
    address public pauser = address(4);
    address public periodConfigUpdater = address(5);
    address public user = address(6);
    address public userB = address(7);
    address public userC = address(8);
    
    uint256 internal constant ONE = 1e18;
    uint256 internal constant PERIOD = 7 days;
    function setUp() public {
        // Deploy mock asset
        asset = new MockERC20();
        
        // Deploy implementation
        FirelightVault implementation = new FirelightVault();
        
        // Prepare init params
        FirelightVault.InitParams memory initParams = FirelightVault.InitParams({
            defaultAdmin: admin,
            limitUpdater: limitUpdater,
            blocklister: blocklister,
            pauser: pauser,
            periodConfigurationUpdater: periodConfigUpdater,
            depositLimit: 1000000 * 10**18,
            periodConfigurationDuration: 7 days
        });
        
        // Encode initialize call
        bytes memory initData = abi.encodeWithSelector(
            FirelightVault.initialize.selector,
            IERC20(address(asset)),
            "Firelight Vault Token",
            "fVault",
            abi.encode(initParams)
        );
        
        // Deploy proxy
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(implementation),
            initData
        );
        
        // Wrap proxy in vault interface
        vault = FirelightVault(address(proxy));
        
        // mint tokens to users
        asset.mint(user, 100000 * 10**18);
        asset.mint(userB, 100000 * 10**18);
        asset.mint(userC, 100000 * 10**18);
    }

    function test_PeriodAtTimestamp_IgnoresArgument() public {
        // @audit Ensure some baseline activity/time
        _deposit(user, 10 * ONE);

        uint256 current = vault.currentPeriod();
        uint256 tsFuture = block.timestamp + 3 * PERIOD; 
        uint256 expected = current + 3;

        // @audit If correct, periodAtTimestamp(tsFuture) should be current+3.
        // @audit but current behavior returns "current" timestamp period
        uint256 reported = vault.periodAtTimestamp(uint48(tsFuture));

        assertTrue(reported != expected, "Expected misreport due to using block.timestamp internally");
    }

    function _deposit(address who, uint256 assets) internal returns (uint256 shares) {
        vm.startPrank(who);
        asset.approve(address(vault), type(uint256).max);
        shares = vault.deposit(assets, who);
        vm.stopPrank();
    }

    function _requestWithdraw(address who, uint256 assets, address receiver) internal returns (uint256 sharesBurned) {
        vm.startPrank(who);
        vault.approve(address(vault), type(uint256).max);
        sharesBurned = vault.withdraw(assets, receiver, who);
        vm.stopPrank();
    }

    function _advanceAtLeast(uint256 secs) internal {
        vm.warp(block.timestamp + secs);
    }

    function _advancePastNextClaimable() internal {
        _advanceAtLeast(15 days);
    }
}


```
