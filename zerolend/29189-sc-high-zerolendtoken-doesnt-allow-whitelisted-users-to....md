# 29189 - \[SC - High] ZeroLendToken doesnt allow whitelisted users to...

Submitted on Mar 10th 2024 at 01:34:08 UTC by @jovi for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29189

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Temporary freezing of funds for at least 1 hour
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The ZeroLendToken doesn't allow whitelisted users to transfer its ZERO tokens.

## Vulnerability Details

When transferring erc20 tokens, the update internal function is called in order to change the balances held by both accounts involved. The ZeroLendToken contract overrides this function implementation to ensure a couple of requirements are met before executing the virtual update function, as can be seen in the following code snippet:

```solidity
function _update(
        address from,
        address to,
        uint256 value
    ) internal virtual override {
        require(!blacklisted[from] && !blacklisted[to], "blacklisted");
        require(!paused && !whitelisted[from], "paused");
        super._update(from, to, value);
    }
```

The issue lies at the second require statement, as it reverts if the from argument is whitelisted. This will make whitelisted users' transfers always revert. If the intent is to allow whitelisted users to be able to transfer while the contract is paused, the requirement statement should be implemented in such a way that does not block whitelisted users' calls.

```solidity
function _update(
        address from,
        address to,
        uint256 value
    ) internal virtual override {
    ...
    if (paused){
	    require(whitelisted[from], "paused");
    }
    ...
    }
```

## Impact Details

Whitelisted users are never able to transfer their tokens.

## References

update internal function implementation at the ZeroLendToken contract: [governance/contracts/ZeroLendToken.sol at a30d8bb825306dfae1ec5a5a47658df57fd1189b · zerolend/governance (github.com)](https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/ZeroLendToken.sol#L56C4-L64C6)

## Proof of concept

Paste the following code snippet inside the test folder:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {console} from "../../lib/forge-std/src/console.sol";
import {StdInvariant} from "../../lib/forge-std/src/StdInvariant.sol";
import {ZeroLend} from "../../ZeroLendToken.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";

contract StakingBonusTest is StdInvariant, Test {

    ZeroLend  zeroToken;
    address user = makeAddr("user");
    address user2 = makeAddr("user2");
    address public configurator = makeAddr("configurator");
    uint256 internal WEEK;
    uint256 internal MAXTIME;

    function setUp() public {
        vm.prank(configurator);
        zeroToken = new ZeroLend();

        vm.prank(configurator);
        zeroToken.togglePause(false);

        vm.prank(configurator);
        zeroToken.toggleWhitelist(address(user), true);

        vm.prank(configurator);
        zeroToken.transfer(address(user), 100 ether);

        vm.prank(configurator);
        zeroToken.transfer(address(user2), 100 ether);

        WEEK = 1 weeks;
        MAXTIME = 365 * 86400;

    }

    function test_pocZeroLendToken() public {

        // attempt to transfer tokens from the whitelisted address and errors
        vm.prank(user);
        vm.expectRevert("paused");
        zeroToken.transfer(address(0x1000), 100 ether);

        // attempt to transfer tokens from the non-whitelisted address and it is successful
        vm.prank(user2);
        zeroToken.transfer(address(0x1000), 100 ether);

       }
}
```

Set up the ZeroLendToken. Whitelist a user. Give him some a balance. Try to transfer from him. Watch it fail.

Run the test with the following command:

```shell
forge test --match-test test_pocZeroLendToken -vvv
```
