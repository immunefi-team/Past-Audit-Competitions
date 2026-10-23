# 29188 - \[SC - Insight] StakingBonuscalculateBonus wrongly utilizes BPS

Submitted on Mar 10th 2024 at 01:21:12 UTC by @jovi for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29188

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/zerolend/governance>

Impacts:

* Protocol insolvency
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The calculateBonus function at the StakingBonus contract utilizes BPS in a pattern that may lead to over-accrual of bonuses.

## Vulnerability Details

Basis points (bps) are a common unit of measure for interest rates and other percentages in finance. One basis point is equal to 1/100 of 1%. In decimal form, one basis point appears as 0.0001 (0.01/100) The usage of basis points is correctly implemented in other parts of the code but not in this one, as it divides the bonusBps variable by 100.

```solidity
function calculateBonus(
        uint256 amount
    ) public view override returns (uint256) {
        uint256 bonus = (amount * bonusBps) / 100;
        // if we don't have enough funds to pay out bonuses, then return 0
        if (zero.balanceOf(address(this)) < bonus) return 0;
        return (amount * bonusBps) / 100;
    }
```

## Impact Details

If the admin decides to set a bonus of 1% then call setBonusBps at the contract, the bonus calculation at the calculateBonus view function will actually return 100%. The reason for that is it will divide the equivalent of 1% in bps (which is 100) by 100. This is concerning as any bonus rate over 1% will yield more value in bonuses than in the actual pending amount. It can generate bonuses of up to 100 times the pending amount.

## References

calculateBonus is called at the onERC721Received function at the StakingBonus contract: [governance/contracts/vesting/StakingBonus.sol at a30d8bb825306dfae1ec5a5a47658df57fd1189b · zerolend/governance (github.com)](https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/vesting/StakingBonus.sol#L68)

calculateBonus implementation at the StakingBonus contract: <https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/vesting/StakingBonus.sol#L89C5-L96C6>

## Proof of concept

Paste the following code snippet inside the test folder:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {console} from "../../lib/forge-std/src/console.sol";
import {StdInvariant} from "../../lib/forge-std/src/StdInvariant.sol";
import {StakingBonus} from "../../vesting/StakingBonus.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MintableERC20 is ERC20 {
    address public owner;

    constructor(string memory name, string memory symbol) ERC20(name, symbol) {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "MintableERC20: caller is not the owner");
        _;
    }

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
}

contract StakingBonusTest is StdInvariant, Test {

    StakingBonus stakingBonus;
    MintableERC20  arbitraryToken;
    address user = makeAddr("user");
    address public configurator = makeAddr("configurator");
    uint256 internal WEEK;
    uint256 internal MAXTIME;

    function setUp() public {
        vm.prank(configurator);
        arbitraryToken = new MintableERC20("Arbitrary Token", "ATKN");


        vm.prank(configurator);
        stakingBonus = new StakingBonus();

        vm.prank(configurator);
        arbitraryToken.mint(address(stakingBonus), 100 ether);

        vm.prank(configurator);
        stakingBonus.init(
            address(arbitraryToken),
            // locker address
            address(0x1000),
            // vested zero address
            address(0x1001),
            // bonusBps 1%, so it should be 100
            100
        );

        WEEK = 1 weeks;
        MAXTIME = 365 * 86400;

    }

    function test_pocStakingBonus() public {

        uint256 bonus = stakingBonus.calculateBonus(1 ether);

        assertEq(bonus, 1 ether);

        vm.prank(configurator);
        // set bonus to 100%. This is equal to 10000 bps.
        stakingBonus.setBonusBps(10000);

        bonus = stakingBonus.calculateBonus(1 ether);

        // notice the bonus is 100 times the expected value
        assertEq(bonus, 100 ether);

       }
}
```

Set up the StakingBonus contract, give him some ZERO balance. Call setBonusBps with 1%, which should be 100 bps. Call calculate bonus with a amount and watch it return the amount. Set bonus to 100%. Watch calculate bonus become 100x the amount.

Run the test with the following command:

```shell
forge test --match-test test_pocStakingBonus -vvv
```
