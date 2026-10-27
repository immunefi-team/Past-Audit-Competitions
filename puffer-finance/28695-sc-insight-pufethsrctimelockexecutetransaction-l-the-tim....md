# 28695 - \[SC - Insight] pufETHsrcTimelockexecuteTransaction - L The tim...

## `pufETH/src/Timelock::executeTransaction()` - L213: The timelock expiry check incorrectly allows for transaction execution AT the final/last block of the timelocked delay represented by `lockedUntil`.

Submitted on Feb 24th 2024 at 04:26:26 UTC by @OxSCSamurai for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28695

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

### Description

### Brief/Intro

## Summary:

The current implementation effectively allows the queued transaction to be executed AT/DURING the last block of the timelock period, which is logically incorrect. We should wait for the timelock period to expire completely before we allow for the queued transaction to be executed.

This means that we should not do `if (block.timestamp < lockedUntil)`, we should instead do `if (block.timestamp <= lockedUntil)`: The buggy code snippet:

```solidity
        if (block.timestamp < lockedUntil) {
            revert Locked(txHash, lockedUntil);
        }
```

### Vulnerability Details

## Context:

See the below evidence which supports my claim. I draw your attention specifically to `"before the lock period expires"`. This error is thrown when a queued/timelocked transaction is attempted BEFORE the lock period EXPIRES:

```solidity
    /**
     * @notice Error to be thrown when a transaction is attempted before the lock period expires
     * @param txHash The keccak256 hash of the locked transaction
     * @param lockedUntil The timestamp when the transaction can be executed
     */
    error Locked(bytes32 txHash, uint256 lockedUntil);
```

In `executeTransaction()` the `lockedUntil` is a local variable that represents the expiry time of the lock period. In other words, if `lockedUntil` was exactly equal to `block.timestamp + 7 days` then the queued timelocked transaction can only be executed AFTER the lock period expired, which effectively is any `block.timestamp > lockedUntil`, because `block.timestamp + 7 days` represents a specific `block.number`, and the queued transaction should only be executed AFTER this specific `block.number`, NOT before it and NOT during it. If we allow this queued transaction to be executed AT `block.timestamp == lockedUntil` then we are effectively allowing the transaction to be executed DURING the timelock expiry block represented by `lockedUntil`. This is not logically correct at all.

We should allow the last/final block of the timelock period to expire completely and only then should we execute the queued transaction during the NEXT block after the block represented by `lockedUntil`. This is the correct way.

### Impact Details

## IMPACT:

I deem this bug to be of at least medium severity due to the potential impacts below and the risks associated with it.

* The operational multisig queues a transaction to pause all unstaking/withdrawals, after a 7 day timelock delay. Some users decide to wait until the last moment to manually/automatically unstake/withdraw their staked funds in order to accumulate max rewards. All should be fine under normal circumstances, but due to this bug, when they try to withdraw their funds at `lockedUntil` block the withdrawal functionality is already frozen, because the pause transaction was included into the `lockedUntil` block first and therefore already executed.
* If the bug was fixed, the pause transaction would only be executed in the block after the `lockedUntil` block, so the users' withdraw transactions would be executed successfully before that.
* We already have a bug where the timelock delay cannot be set to 7 days, which should be possible but the bug prevents this, and now this bug where the delayed transaction can already be executed during the final block of the timelock delay, i.e. at/during `lockedUntil` block. Not too good for trust in the protocol.
* Not sure exactly how this could benefit an attacker/blackhat, but if he knows about this bug while no one else knows about it, how could he take advantage of this?...He would know that the delayed transaction would be executed at time N, but everyone else would expect the transaction to get executed at time N+1, if you see what I mean.

Below I describe an example of how this type of bug could affect a user. Not saying this is relevant to your protocol, but it demonstrates the potential risk in such scenarios:

Hypothetical scenario:

> In a Liquid Restaking (LRT) DeFi protocol, a change is queued for execution via a 7-day timelock. This change involves adjusting the parameters governing the penalty for early unstaking. Currently, the penalty for early unstaking is relatively low, encouraging users to unstake their tokens without significant loss if needed.

> However, the queued change intends to increase the penalty for early unstaking substantially. After the 7-day timelock expires, this change will take effect, imposing a much higher penalty on users who unstake their tokens before a certain period, let's say 30 days.

> Now, let's imagine a user who needs to access their staked tokens urgently for personal reasons. If this user fails to unstake their tokens before the timelock expires and the new penalty mechanism is implemented, they would suffer negative consequences.

> After the change is executed, the user would face a significantly higher penalty for unstaking their tokens early. This could result in a substantial loss of value compared to unstaking before the change takes effect. As a result, the user might find themselves in a situation where accessing their tokens becomes much more costly than anticipated, leading to financial inconvenience and potential dissatisfaction with the protocol.

> This scenario highlights the importance for users to stay informed about impending changes in the protocol and take timely action to avoid negative consequences, such as unexpected penalties, that may arise from not unstaking their tokens before the implementation of such changes.

> If the user decides to wait until the very last moment, i.e. the `lockedUntil` block to unstake their tokens, i.e. the block where the timelock period expires, and if the user's unstake transaction is included into the block AFTER the transaction for the new unstake penalty change which gets executed in this same block due to the bug, then the user's unstake transaction will be penalised according to the new change. This will catch the user unexpectedly because the user checked the protocol and saw that the new change should only be executed AFTER the timelock period expires, which means AFTER `lockedUntil`. So the user planned accordingly but did not know about the bug...

Surely there are many other scenarios where this type of bug can cause issues for users/investors, or for integrating protocols, etc.

## IMPACTS IN SCOPE:

* contract fails to deliver promised returns >>> should be self evident from above descriptions

Not 100% sure about the following two impacts:

* temporary freezing of funds >= 1hr >>> the validity of this impact is at the discretion of the protocol team. I added this here because if one or more users wait until the `lockedUntil` block to withdraw/unstake their funds, and the timelock delayed transaction is supposed to pause all withdrawals/functions, then effectively their funds will be temporarily frozen, yes? No idea if this counts. If it does, and if you would like a PoC for same, let me know\...
* griefing attack: this isnt an attack, its the bug that griefs users who wait until `lockedUntil` block to do something. again, the protocol team can add their two cents here and let me know what they think.

### References

<https://github.com/PufferFinance/pufETH/blob/3e76d02c7b54323d347c8277327d3877bab591f5/src/Timelock.sol#L213>

### Proof of Concept

## PoC:

For the buggy function I added two additional lines of code to test:

```solidity
        if (block.timestamp < lockedUntil) {
        //if (block.timestamp <= lockedUntil) { /// @audit added for PoC/testing purposes
        //if (block.timestamp == lockedUntil) { /// @audit added for PoC/testing purposes
            revert Locked(txHash, lockedUntil);
        }
```

Using the following test function which I've modified for my PoC tests: `pufETH/test/unit/Timelock.t.sol::test_set_delay_queued()`:

```solidity
    function test_set_delay_queued() public {
        vm.startPrank(timelock.OPERATIONS_MULTISIG());

        bytes memory callData = abi.encodeCall(Timelock.setDelay, (15 days));

        assertTrue(timelock.delay() != 15 days, "initial delay");

        uint256 operationId = 1234;

        bytes32 txHash = timelock.queueTransaction(address(timelock), callData, operationId);

        uint256 lockedUntil = block.timestamp + timelock.delay();

        //vm.expectRevert(abi.encodeWithSelector(Timelock.Locked.selector, txHash, lockedUntil));
        //timelock.executeTransaction(address(timelock), callData, operationId);

        //vm.warp(lockedUntil + 1);
        vm.warp(lockedUntil); /// @audit added for PoC/testing purposes

        timelock.executeTransaction(address(timelock), callData, operationId);

        //assertEq(timelock.delay(), 15 days, "updated the delay");
    }
```

In above test function I test both the following:

```solidity
        //vm.warp(lockedUntil + 1); /// @audit should be allowed to execute timelocked tx at this point.
        vm.warp(lockedUntil); /// @audit should NOT be allowed to execute timelocked transaction at this point.
```

For the tests I use the following forge command: `ETH_RPC_URL=https://mainnet.infura.io/v3/YOUR_KEY forge test --match-test test_set_delay_queued -vvvv`

## TEST 1:

## Test using `vm.warp(lockedUntil + 1);`:

This wont catch the bug, and is the default setting in the test function above. Wont revert for any of below cases.

1. test for `if (block.timestamp < lockedUntil) {` >>> `block.timestamp` == `lockedUntil` is valid, dont revert
2. test for `if (block.timestamp <= lockedUntil) {` >>> `block.timestamp` == `lockedUntil` is invalid, revert
3. test for `if (block.timestamp == lockedUntil) {` >>> `block.timestamp` == `lockedUntil` is invalid, revert

RESULTS:

1. test for `if (block.timestamp < lockedUntil) {`:

```solidity
Traces:
  [42274] TimelockTest::test_set_delay_queued()
    ├─ [227] Timelock::OPERATIONS_MULTISIG() [staticcall]
    │   └─ ← operationsMultisig: [0x78dE5808728A273648A7D301D7767C4fd5Dc0fF6]
    ├─ [0] VM::startPrank(operationsMultisig: [0x78dE5808728A273648A7D301D7767C4fd5Dc0fF6])
    │   └─ ← ()
    ├─ [2404] Timelock::delay() [staticcall]
    │   └─ ← 691200 [6.912e5]
    ├─ [27548] Timelock::queueTransaction(Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, 1234)
    │   ├─ emit TransactionQueued(txHash: 0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, target: Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], callData: 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, operationId: 1234, lockedUntil: 691201 [6.912e5])
    │   └─ ← 0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741
    ├─ [404] Timelock::delay() [staticcall]
    │   └─ ← 691200 [6.912e5]
    ├─ [0] VM::warp(691202 [6.912e5])
    │   └─ ← ()
    ├─ [10037] Timelock::executeTransaction(Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, 1234)
    │   ├─ [4721] Timelock::setDelay(1296000 [1.296e6])
    │   │   ├─ emit DelayChanged(oldDelay: 691200 [6.912e5], newDelay: 1296000 [1.296e6])
    │   │   └─ ← ()
    │   ├─ emit TransactionExecuted(txHash: 0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, target: Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], callData: 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, operationId: 1234)
    │   └─ ← true, 0x
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.63ms

Ran 1 test suite in 6.63ms: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

2. test for `if (block.timestamp <= lockedUntil) {`:

```solidity
Traces:
  [42272] TimelockTest::test_set_delay_queued()
    ├─ [227] Timelock::OPERATIONS_MULTISIG() [staticcall]
    │   └─ ← operationsMultisig: [0x78dE5808728A273648A7D301D7767C4fd5Dc0fF6]
    ├─ [0] VM::startPrank(operationsMultisig: [0x78dE5808728A273648A7D301D7767C4fd5Dc0fF6])
    │   └─ ← ()
    ├─ [2404] Timelock::delay() [staticcall]
    │   └─ ← 691200 [6.912e5]
    ├─ [27548] Timelock::queueTransaction(Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, 1234)
    │   ├─ emit TransactionQueued(txHash: 0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, target: Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], callData: 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, operationId: 1234, lockedUntil: 691201 [6.912e5])
    │   └─ ← 0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741
    ├─ [404] Timelock::delay() [staticcall]
    │   └─ ← 691200 [6.912e5]
    ├─ [0] VM::warp(691202 [6.912e5])
    │   └─ ← ()
    ├─ [10034] Timelock::executeTransaction(Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, 1234)
    │   ├─ [4721] Timelock::setDelay(1296000 [1.296e6])
    │   │   ├─ emit DelayChanged(oldDelay: 691200 [6.912e5], newDelay: 1296000 [1.296e6])
    │   │   └─ ← ()
    │   ├─ emit TransactionExecuted(txHash: 0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, target: Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], callData: 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, operationId: 1234)
    │   └─ ← true, 0x
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.11ms

Ran 1 test suite in 6.11ms: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

3. test for `if (block.timestamp == lockedUntil) {`:

```solidity
Traces:
  [42272] TimelockTest::test_set_delay_queued()
    ├─ [227] Timelock::OPERATIONS_MULTISIG() [staticcall]
    │   └─ ← operationsMultisig: [0x78dE5808728A273648A7D301D7767C4fd5Dc0fF6]
    ├─ [0] VM::startPrank(operationsMultisig: [0x78dE5808728A273648A7D301D7767C4fd5Dc0fF6])
    │   └─ ← ()
    ├─ [2404] Timelock::delay() [staticcall]
    │   └─ ← 691200 [6.912e5]
    ├─ [27548] Timelock::queueTransaction(Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, 1234)
    │   ├─ emit TransactionQueued(txHash: 0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, target: Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], callData: 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, operationId: 1234, lockedUntil: 691201 [6.912e5])
    │   └─ ← 0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741
    ├─ [404] Timelock::delay() [staticcall]
    │   └─ ← 691200 [6.912e5]
    ├─ [0] VM::warp(691202 [6.912e5])
    │   └─ ← ()
    ├─ [10034] Timelock::executeTransaction(Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, 1234)
    │   ├─ [4721] Timelock::setDelay(1296000 [1.296e6])
    │   │   ├─ emit DelayChanged(oldDelay: 691200 [6.912e5], newDelay: 1296000 [1.296e6])
    │   │   └─ ← ()
    │   ├─ emit TransactionExecuted(txHash: 0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, target: Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], callData: 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, operationId: 1234)
    │   └─ ← true, 0x
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.18ms

Ran 1 test suite in 6.18ms: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## TEST 2:

## Test using `vm.warp(lockedUntil);`:

This will catch the bug, and was added to the test function by myself.

## We expect 2) and 3) to revert, proving the bug.

1. test for `if (block.timestamp < lockedUntil) {` >>> `block.timestamp` == `lockedUntil` is valid, dont revert
2. test for `if (block.timestamp <= lockedUntil) {` >>> `block.timestamp` == `lockedUntil` is invalid, revert
3. test for `if (block.timestamp == lockedUntil) {` >>> `block.timestamp` == `lockedUntil` is invalid, revert

RESULTS:

1. test for `if (block.timestamp < lockedUntil) {`:

```solidity
Traces:
  [42216] TimelockTest::test_set_delay_queued()
    ├─ [227] Timelock::OPERATIONS_MULTISIG() [staticcall]
    │   └─ ← operationsMultisig: [0x78dE5808728A273648A7D301D7767C4fd5Dc0fF6]
    ├─ [0] VM::startPrank(operationsMultisig: [0x78dE5808728A273648A7D301D7767C4fd5Dc0fF6])
    │   └─ ← ()
    ├─ [2404] Timelock::delay() [staticcall]
    │   └─ ← 691200 [6.912e5]
    ├─ [27548] Timelock::queueTransaction(Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, 1234)
    │   ├─ emit TransactionQueued(txHash: 0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, target: Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], callData: 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, operationId: 1234, lockedUntil: 691201 [6.912e5])
    │   └─ ← 0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741
    ├─ [404] Timelock::delay() [staticcall]
    │   └─ ← 691200 [6.912e5]
    ├─ [0] VM::warp(691201 [6.912e5])
    │   └─ ← ()
    ├─ [10037] Timelock::executeTransaction(Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, 1234)
    │   ├─ [4721] Timelock::setDelay(1296000 [1.296e6])
    │   │   ├─ emit DelayChanged(oldDelay: 691200 [6.912e5], newDelay: 1296000 [1.296e6])
    │   │   └─ ← ()
    │   ├─ emit TransactionExecuted(txHash: 0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, target: Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], callData: 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, operationId: 1234)
    │   └─ ← true, 0x
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.36ms

Ran 1 test suite in 5.36ms: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

2. test for `if (block.timestamp <= lockedUntil) {`:

```solidity
    ├─ [1386] Timelock::executeTransaction(Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, 1234)
    │   └─ ← Locked(0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, 691201 [6.912e5])
    └─ ← Locked(0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, 691201 [6.912e5])

Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 5.96ms

Ran 1 test suite in 5.96ms: 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/unit/Timelock.t.sol:TimelockTest
[FAIL. Reason: Locked(0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, 691201 [6.912e5])] test_set_delay_queued() (gas: 43523)

Encountered a total of 1 failing tests, 0 tests succeeded
```

3. test for `if (block.timestamp == lockedUntil) {`:

```solidity
    ├─ [1386] Timelock::executeTransaction(Timelock: [0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9], 0xe177246e000000000000000000000000000000000000000000000000000000000013c680, 1234)
    │   └─ ← Locked(0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, 691201 [6.912e5])
    └─ ← Locked(0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, 691201 [6.912e5])

Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 6.06ms

Ran 1 test suite in 6.06ms: 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/unit/Timelock.t.sol:TimelockTest
[FAIL. Reason: Locked(0x5d875d407f0be15066debfe46b854bb8d0ddf8cc03e75286374bd0c70e235741, 691201 [6.912e5])] test_set_delay_queued() (gas: 43523)

Encountered a total of 1 failing tests, 0 tests succeeded
```

## Recommendation:

```diff
-       if (block.timestamp < lockedUntil) {
+       if (block.timestamp <= lockedUntil) {
            revert Locked(txHash, lockedUntil);
        }
```
