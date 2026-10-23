# 26066 - \[SC - Insight] Timelock eta variable can be set further than i...

Submitted on Nov 24th 2023 at 01:24:49 UTC by @p4rsely for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26066

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xf2991507952d9594e71a44a54fb19f3109d213a5#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

In the TimeLock contract when queueing a new transaction, the `eta` parameter is not checked to be less than the maximum delay time constant. This allows the `eta` to be set further than 365 days in the future.

This is not a problem in itself, however it negates having a maximum delay constant value of 365 days as the `eta` can be any value greater than the current timestamp plus the delay value. This could cause a delay in upgrades as it may not be immediately obvious/caught that the `eta` is too far in the future. The transaction can always be cancelled but may cause lost time for the upgrade.

The current code is below: <https://github.com/degatedev/protocols/blob/degate\\_mainnet/packages/loopring\\_v3/contracts/thirdparty/timelock/Timelock.sol#L63-L72>

```
    function queueTransaction(address target, uint value, string memory signature, bytes memory data, uint eta) public returns (bytes32) {
        require(msg.sender == admin, "Timelock::queueTransaction: Call must come from admin.");
        require(eta >= getBlockTimestamp().add(delay), "Timelock::queueTransaction: Estimated execution block must satisfy delay.");

        bytes32 txHash = keccak256(abi.encode(target, value, signature, data, eta));
        queuedTransactions[txHash] = true;

        emit QueueTransaction(txHash, target, value, signature, data, eta);
        return txHash;
    }

```

## Impact

Thus could lead to an unforeseen delay in the upgrade process.

## Risk Breakdown

Difficulty to Exploit: Easy

## Recommendation

It can be considered to let the contract set the ETA as the value of `block.timestamp` plus the delay. This approach would allow for more predictable upgrade schedules.

```
    function queueTransaction(address target, uint value, string memory signature, bytes memory data,uint) public returns (bytes32) {
        require(msg.sender == admin, "Timelock::queueTransaction: Call must come from admin.");
        uint256 eta = block.timestamp.add(delay);

        // this check below can then be removed so i ahve commented it out
        //require(eta >= getBlockTimestamp().add(delay), "Timelock::queueTransaction: Estimated execution block must satisfy delay.");

        bytes32 txHash = keccak256(abi.encode(target, value, signature, data, eta));
        queuedTransactions[txHash] = true;

        emit QueueTransaction(txHash, target, value, signature, data, eta);
        return txHash;
    }
```

## References

Contracts:

<https://github.com/degatedev/protocols/blob/degate\\_mainnet/packages/loopring\\_v3/contracts/thirdparty/timelock/Timelock.sol#L63-L72>

## Proof of concept

## PoC

Please copy/paste the code below into a file in the test directory of a foundry project called `TansactionQueueTest.t.sol`

Please run the test with a fork of the mainnet `forge test --fork-url {YOU_RPC_PROVIDER} --match-test test_queueTransaction -vv`

I also had to add this line below to the foundry.toml file `evm_version = "shanghai"`

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.19;

import {Test, console} from "forge-std/Test.sol";
import "src/thirdparty/timelock/Timelock.sol";


contract TansactionQueueTest is Test {
    Timelock public _timeLock;
    address multisig = address(0x2028834B2c0A36A918c10937EeA71BE4f932da52);

    function setUp() public {
        _timeLock = Timelock(payable(0xf2991507952d9594E71A44A54fb19f3109D213A5));

    }

    function test_queueTransaction() public {
        uint256 distantETA = block.timestamp + 700 days;
        // now queue the transaction 700 days in advance
        console.log("[i] 700 days int value is : ", distantETA);
        console.log("[i] The delay is : ", _timeLock.delay());
        // prank as multisig admin
        vm.prank(multisig);
        bytes32 hashResp = _timeLock.queueTransaction(address(123), 0, "signature", "data", distantETA);
        console.log("[i] The returned hash value is : ");
        console.logBytes32(hashResp);
        console.log("[i] The transaction was successfully queued at an ETA of 700 days plus delay");

    }
}

```
