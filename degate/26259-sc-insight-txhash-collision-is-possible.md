# 26259 - \[SC - Insight] txHash collision is possible

Submitted on Nov 29th 2023 at 18:23:56 UTC by @Madalad for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26259

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xf2991507952d9594e71a44a54fb19f3109d213a5#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

If the admin wishes to make the same call more than once with the same eta, this is impossible due to the fact they would share the same `txHash`. The admin would have to queue the transaction, wait at least 45 days, execute it, then queue it again and wait a further 45 days.

## Impact

Queueing two similar transactions causes the first to be overwritten, potentially deceiving the admin and/or users.

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: CVSS2 Score:

## Recommendation

While it is possible to workaround this issue fairly simply, e.g. altering the `eta` to differ by one second, implementing some form of nonce in the hash calculation is conventional and provides the least amount of confusion/inconvenience.

## References

## Proof of concept

See below the proof of concept in the form of a foundry test file.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

pragma experimental ABIEncoderV2;

import "forge-std/Test.sol";
import "../../../src/contracts/timelock/Timelock.sol";

contract CollisionTest is Test {
    Timelock public timelock;

    address public admin = makeAddr("admin");
    uint256 public delay = 45 days;

    function setUp() public {
        vm.startPrank(admin);
        timelock = new Timelock({
            admin_: admin,
            delay_: delay
        });
        vm.stopPrank();
    }

    function testCollision() public {
        // Make transaction
        address target = makeAddr("target");
        uint256 value = 0;
        string memory signature = "";
        bytes memory data = abi.encode("data");
        uint256 eta = block.timestamp + delay;

        // Queue transaction twice
        vm.startPrank(admin);
        timelock.queueTransaction(target, value, signature, data, eta);
        timelock.queueTransaction(target, value, signature, data, eta);

        // Execute first transaction
        vm.warp(block.timestamp + delay);
        timelock.executeTransaction(target, value, signature, data, eta);

        // Cannot execute second transaction
        vm.warp(block.timestamp + delay);
        vm.expectRevert("Timelock::executeTransaction: Transaction hasn't been queued.");
        timelock.executeTransaction(target, value, signature, data, eta);
    }
}
```

Output:

```
Running 1 test for test/timelock/PoCs/Collision.t.sol:CollisionTest
[PASS] testCollision() (gas: 57777)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 109.99ms
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
