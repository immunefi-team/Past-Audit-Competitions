# 29137 - \[SC - High] ZeroLend token is not behaving properly while c...

Submitted on Mar 8th 2024 at 09:38:30 UTC by @dontonka for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29137

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* ZeroLendToken not properly behaving in prelaunch phase

## Description

## Brief/Intro

`ZeroLendToken` should allow to operate with Whitelisted users even if contract is paused, which is not the case in the current implementation which seems to warrant `Low` severity.

## Vulnerability Details

The current condition is inaccurate.

## Impact Details

Whitelisted user will not be able to use the contract while the contract is paused.

## Recommendation

Apply the following changes.

```diff
    function _update(
        address from,
        address to,
        uint256 value
    ) internal virtual override {
        require(!blacklisted[from] && !blacklisted[to], "blacklisted");
-       require(!paused && !whitelisted[from], "paused");
+       require(!paused || (whitelisted[from] || whitelisted[to]), "paused");         
         super._update(from, to, value);
    }
```

## Proof of Concept

Run the following command to create the fresh testing environnement based on Foundry.

```
foundryup
forge init zerolend
cd zerolend 
forge install OpenZeppelin/openzeppelin-contracts
Create `ZeroLendToken.sol` in src
Create `ZeroLendToken.t.sol` in test
rm src/Counter.sol
rm test/Counter.t.sol
forge test --match-test
```

```
[PASS] test_owner_transfer_to_bl() (gas: 15619)
[PASS] test_owner_transfer_to_normal() (gas: 17743)
[FAIL. Reason: revert: paused] test_owner_transfer_to_wl() (gas: 14356)
Suite result: FAILED. 2 passed; 1 failed; 0 skipped; finished in 1.22ms (231.74µs CPU time)

Ran 1 test suite in 385.67ms (1.22ms CPU time): 2 tests passed, 1 failed, 0 skipped (3 total tests)

Failing tests:
Encountered 1 failing test in test/ZeroLendToken.t.sol:ZeroLendTokenTest
[FAIL. Reason: revert: paused] test_owner_transfer_to_wl() (gas: 14356)
```

Apply the recommended fix and test will pass as follow.

```
Ran 3 tests for test/ZeroLendToken.t.sol:ZeroLendTokenTest
[PASS] test_owner_transfer_to_bl() (gas: 15619)
[PASS] test_owner_transfer_to_normal() (gas: 22152)
[PASS] test_owner_transfer_to_wl() (gas: 48299)
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 1.31ms (312.34µs CPU time)
```

**ZeroLendToken.sol**, the original file.

**ZeroLendToken.t.sol**

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {ZeroLend} from "../src/ZeroLendToken.sol";

contract ZeroLendTokenTest is Test {
    ZeroLend public zero;
    address alice = address(1); //WL
    address bob = address(2); //BL
    address tom = address(3); // normal

    function setUp() public {
        zero = new ZeroLend();
        zero.toggleWhitelist(alice, true);
        zero.toggleBlacklist(bob, true);

        // prelaunch - contract is paused!

        // confirm everything is accurate
        assertEq(zero.whitelisted(alice), true);
        assertEq(zero.blacklisted(alice), false);
        assertEq(zero.whitelisted(bob), false);
        assertEq(zero.blacklisted(bob), true);
        assertEq(zero.blacklisted(tom), false);
        assertEq(zero.whitelisted(tom), false);
        assertEq(zero.whitelisted(address(this)), false);
        assertEq(zero.blacklisted(address(this)), false);
    }

    function test_owner_transfer_to_bl() public {
        vm.expectRevert(bytes("blacklisted"));
        zero.transfer(bob, 1);
    }

    function test_owner_transfer_to_normal() public {
        vm.expectRevert(bytes("paused"));
        zero.transfer(tom, 1);
    }

    function test_owner_transfer_to_wl() public {
        zero.transfer(alice, 1); // FAILs bc condition is wrong (bug)
    }
}
```
