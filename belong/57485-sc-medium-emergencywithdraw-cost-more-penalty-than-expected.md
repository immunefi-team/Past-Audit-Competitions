# 57485 sc medium emergencywithdraw cost more penalty than expected

**Submitted on Oct 26th 2025 at 16:09:18 UTC by @ox9527 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57485
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief/Intro

The penalty cost in `emergencyWithdraw` / `emergencyRedeem` depends on the user’s withdrawal amount. Each deposit must individually wait for its own unlock period, and there is no method for a user to unlock already locked shares.

Assume the following scenario:

* Bob deposits 5e18 for the first time.
* After `staking.minStakePeriod() + 1` time has passed, Bob deposits another 5e18.
* At this point, the first 5e18 is already unlocked, while the second deposit remains locked.

When Bob performs `emergencyWithdraw(10e18)`, the penalty is calculated based on the total withdrawal amount, resulting in a 1e18 penalty instead of the expected 0.5e18.

### Vulnerability Details

The problematic function in Staking.sol:

{% code title="Staking.sol (excerpt)" %}

```
```

{% endcode %}

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

This implementation does not consider whether individual stakes are unlocked or locked; it removes arbitrary shares across the user's stakes when withdrawing, which mixes locked and unlocked stakes and causes the penalty calculation for emergency withdraws to be based on the total withdrawal amount rather than only the locked portion.

## Impact Details

{% hint style="warning" %}
User funds can be permanently frozen or incur larger-than-expected penalties due to locked shares being incorrectly considered when removing shares for emergency withdrawals.
{% endhint %}

## Proof of Concept

<details>

<summary>PoC Test (expand to view)</summary>

{% code title="tests/PoC.sol" %}

```
```

{% endcode %}

\`\`\`solidity // SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol"; import {Staking} from "../contracts/v2/periphery/Staking.sol"; import "node\_modules/@openzeppelin/contracts/token/ERC20/ERC20.sol"; import "node\_modules/@openzeppelin/contracts/token/ERC20/IERC20.sol"; import "forge-std/console2.sol";

contract Long is ERC20 { constructor() ERC20("LONG Mock", "Long") { \_mint(msg.sender, 1\_000\_000\_000\_000 \* 10 \*\* decimals()); }

```
function mint(address to, uint256 amount) public {
    _mint(to, amount);
}
```

} contract StakingTest is Test {

```
Staking public staking;
Long public long;
function setUp() public {
    staking = new Staking();
    long = new Long();
    staking.initialize(address(this), address(this), address(long));

}

//PoC3. Emergency withdraw not consider the already unlocked assets.
//Assume user have 10e18 totaly assets , 5e18->locked,5e18->unlocked.
function test_POC_3() public {
    address alice = address(0x1001);
    address bob = address(0x1002);
    long.mint(alice,10e18);
    long.mint(bob,10e18);

    vm.startPrank(alice);
    long.approve(address(staking), 10e18);
    staking.deposit(10e18,alice);
    vm.stopPrank();

    vm.startPrank(bob);
    long.approve(address(staking), 10e18);
    staking.deposit(5e18,bob);
    vm.stopPrank();

    vm.warp(block.timestamp + staking.minStakePeriod() + 1);

    //bob stake another 5e18.
    vm.startPrank(bob);
    staking.deposit(5e18,bob);

    //bob withdraw.
    staking.emergencyWithdraw(10e18, bob, bob);

    //10e18 - 10e18 * 1% = 9e18.
    console2.log("bob get assets:",long.balanceOf(bob));
}
```

}

```
</details>

<details>
<summary>PoC Output (expand to view)</summary>

```

\[PASS] test\_POC\_3() (gas: 395902) Logs: bob get assets: 9000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.84ms (2.07ms CPU time)

Ran 1 test suite in 142.64ms (6.84ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```
</details>

## References

(Original target file) https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol

--- 

Notes:
- The report highlights that stake removal doesn't respect per-deposit unlock state; thus emergency penalty calculations can be larger than intended when mixing locked and unlocked deposits.
- No remediation or code changes are proposed in the original report; this page preserves the original content as submitted.
```

</details>
