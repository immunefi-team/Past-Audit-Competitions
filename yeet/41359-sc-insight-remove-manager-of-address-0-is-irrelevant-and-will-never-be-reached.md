# #41359 \[SC-Insight] Remove Manager of Address 0 is irrelevant and will never be reached

**Submitted on Mar 14th 2025 at 09:48:30 UTC by @styphoiz for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41359
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**

## Description

## Brief/Intro

The check for removing a manager with address 0 is unnecessary because the process does not allow adding a manager with address 0 in the first place.

## Vulnerability Details

The code contains a check for address 0 in the `removeManager` function, but this condition can never be met since the `addManager` function explicitly prevents adding address 0 as a manager.

## Impact Details

The following code demonstrates why the check in `removeManager` is redundant:

In the `addManager` function, there is a validation that rejects address 0:

```
    function addManager(address _manager) external override onlyOwner {
        require(!managers[_manager], "Manager already exists");
        require(_manager != address(0), "Invalid address");
        managers[_manager] = true;
    }
```

As a result, address 0 can never be added to the managers mapping.

Consequently, the following check in `removeManager` serves no purpose, as address 0 cannot exist in the `managers` mapping:

```
    function removeManager(address _manager) external override onlyOwner {
        require(managers[_manager], "Manager does not exist");
        require(_manager != address(0), "Invalid address");
        managers[_manager] = false;
    }
```

Therefore, this line in `removeManager` is redundant and can be safely removed as this will already be rejected as `Manager does not exist`

```
        require(_manager != address(0), "Invalid address");
```

## References

## Proof of Concept

## Proof of Concept

See below PoC showcasing this test, PoC has been updated from the StakeV2.test.sol test case.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "forge-std/console2.sol";
import "../src/StakeV2.sol";
import {MockERC20} from "./mocks/MockERC20.sol";
import {MockWETH} from "./mocks/MockWBERA.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "./mocks/SimpleZapperMock.sol";

contract KodiakVaultV1 {
    IERC20 public token0;
    IERC20 public token1;

    constructor(IERC20 _token0, IERC20 _token1) {
        token0 = _token0;
        token1 = _token1;
    }
}

abstract contract StakeV2_BaseTest {
    StakeV2 public stakeV2;
    MockERC20 public token;
    MockWETH public wbera;
    SimpleZapperMock public mockZapper;

    function setUp() public virtual {
        token = new MockERC20("MockERC20", "MockERC20", 18);
        wbera = new MockWETH();
        address owner = address(this);
        address manager = address(this);
        mockZapper = new SimpleZapperMock(token, wbera);
        stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));
    }

    function test() public {}
}



contract StakeV2_Manager is Test {
    Manager private _managerContract;

    function setUp() public {
        _managerContract = new Manager(address(0x00a), address(0x00a));
    }

    function test_removeManager() public {
        vm.startPrank(address(0x00a));
        assertFalse(_managerContract.managers(address(0x00c)));
        _managerContract.removeManager(address(0x0));
    }
}
```
