# 29467 - \[SC - Low] RewardTimelockexecuteRewardTransaction - L Inco...

## `RewardTimelock::executeRewardTransaction()` - L112: Incorrect comparison operator results in not being able to execute reward tx at the very last second of tx expiration timestamp.

Submitted on Mar 20th 2024 at 00:07:36 UTC by @OxSCSamurai for [Boost | Immunefi Arbitration](https://immunefi.com/bounty/immunefiarbitration-boost/)

Report ID: #29467

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/immunefi-team/vaults/blob/main/src/RewardTimelock.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Temporary freezing of funds

### Description

### Brief/Intro

`RewardTimelock::executeRewardTransaction()` - L112: Incorrect comparison operator results in not being able to execute reward tx at the very last second of tx expiration timestamp.

Disclaimer note:

* I'm only including this bug report because it might be of some value to Immunefi team. Initially I got excited because I thought hey I've got a legit bug here, but after deep-diving this and doing multiple PoC test runs I leveled up my insight and understanding of block timestamps, block numbers, and how incorrect use of comparison operators can or cannot affect these... So I dont expect you to validate this report, but who knows maybe you agree with my assessment below.

The bug allows for premature expiration of reward transaction at least 1 second too early, and undermines whitehat experience by delaying reward transfer if the transaction reverts.

### Vulnerability Details

SUMMARY:

* There are two possible approaches: -- A: something expires AFTER a specific time period, i.e. expires AFTER 60 seconds, i.e. AT 60:00:001 and onwards. OR -- B: something expires AT a specific time, i.e. expires AT exactly 60 seconds onwards, i.e. AT 60:00:000 and onwards.
* If the `txData.expiration` uses approach B, according to intended protocol logic, then this bug report is invalid.
* But if `txData.expiration` uses approach A, according to intended protocol logic, then this bug report is valid.
* My assumption is it's approach A, which means the current system's implementation of transaction expiration presents an issue. When setting a 60-second expiration period, as example, transactions are prematurely expiring AT the 60 second mark, instead of at > 60 seconds only. For instance, if a cooldown period ends at precisely 12:00pm and reward transaction expiry is set to 60 seconds later, i.e. 12:01:00:000, the tx should still be valid until that exact moment, and only become expired at 12:01:00:001 onwards. This can result in delays in rewards transfers because the txs will revert.

### Impact Details

IMPACT:

* Likelihood: High, Impact: Low = Medium severity.
* Potential temporary freezing of rewards for the duration of the next cooldown period + expiration, causing frustration for whitehats.

Code snippet in function `executeRewardTransaction()` where the expiration check is implemented:

```solidity
require(
    txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration > block.timestamp,
    "RewardTimelock: transaction is expired"
);
```

* Additionally, are we saying that the reward transaction is valid and non-expired during this entire period below(after cooldown), or only up to excluding the last second of this period below?
* `txData.queueTimestamp + txData.cooldown + txData.expiration` == 12:01:00:000. Should the tx be considered expired at:
* `> 12:01:00:000` ? OR
* `>= 12:01:00:000` ?

### References

Add any relevant links to documentation or code

### Proof of Concept

PoC:

Ok, so I've managed to modify the existing tests, and `RewardTimelock.sol` contract so that it returns the exact difference in seconds between the `block.timestamp` of the reward tx execution, and the tx expiration timestamp.

So IF this is a bug, then the PoC proves it 100%. If it's not a bug but intended protocol functionality, then the PoC could be something interesting/valuable to check out. Regardless, I had fun taking myself through this process.

The test function I used: <https://github.com/immunefi-team/vaults/blob/49c1de26cda19c9e8a4aa311ba3b0dc864f34a25/test/foundry/RewardTimelock.t.sol#L152-L196>

```solidity
    function testTransactionExpiredRevertsExecution() public {
        uint256 value = 1 ether;
        vm.deal(address(vault), value);

        // set right permissions on moduleGuard
        vm.startPrank(protocolOwner);
        moduleGuard.setTargetAllowed(address(vaultDelegate), true);
        moduleGuard.setAllowedFunction(address(vaultDelegate), vaultDelegate.sendReward.selector, true);
        moduleGuard.setDelegateCallAllowedOnTarget(address(vaultDelegate), true);
        vm.stopPrank();

        uint256 nonce = rewardTimelock.vaultTxNonce(address(vault));
        bytes32 txHash = rewardTimelock.getQueueTransactionHash(address(this), 2000, address(vault), nonce);

        // Mock vaultIsInArbitration
        vm.mockCall(
            address(arbitration),
            abi.encodeCall(arbitration.vaultIsInArbitration, (address(vault))),
            abi.encode(true)
        );

        vm.expectEmit(true, true, true, true);
        emit TransactionQueued(txHash, address(this), address(vault), 2000);
        _sendTxToVault(
            address(rewardTimelock),
            0,
            abi.encodeCall(rewardTimelock.queueRewardTransaction, (address(this), 2000)),
            Enum.Operation.Call
        );

        assertEq(rewardTimelock.vaultTxNonce(address(vault)), nonce + 1);

        vm.warp(block.timestamp + rewardTimelock.txCooldown() + rewardTimelock.txExpiration() - 0 seconds);
        assertFalse(rewardTimelock.canExecuteTransaction(txHash));

        Rewards.ERC20Reward[] memory erc20Rewards = new Rewards.ERC20Reward[](0);

        _sendTxToVault(
            address(rewardTimelock),
            0,
            abi.encodeCall(rewardTimelock.executeRewardTransaction, (txHash, 0, erc20Rewards, 1 ether, 50_000)),
            Enum.Operation.Call,
            true
        );
    }
```

My modifications to the `executeRewardTransaction()` function: <https://github.com/immunefi-team/vaults/blob/49c1de26cda19c9e8a4aa311ba3b0dc864f34a25/src/RewardTimelock.sol#L98-L139>

```solidity
        uint256 expirationTimestamp = txData.queueTimestamp + txData.cooldown + txData.expiration; /// @audit added for PoC/testing purposes
        int256 timeDifferenceRewardTxBlockvsExpirationBlock = int256(block.timestamp) - int256(expirationTimestamp); /// @audit added for PoC/testing purposes
        emit CurrentTransactionBlockNumber(block.number, block.timestamp, expirationTimestamp, timeDifferenceRewardTxBlockvsExpirationBlock); /// @audit added for PoC/testing purposes

        require(
            txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration > block.timestamp, 
            //txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration >= block.timestamp, /// @audit added for PoC/testing purposes
            "RewardTimelock: transaction is expired"
        );
```

As well as the event from above that I defined in the `IRewardTimelockEvents` interface:

```solidity
event CurrentTransactionBlockNumber(uint256 blocknumber, uint256 timestamp, uint256 expirationTimestamp, int256 timeDifferenceRewardTxBlockvsExpirationBlock);
```

## PoC TESTS:

Test command: `forge test --contracts src/RewardTimelock.sol --mt testTransactionExpiredRevertsExecution -vvv`

Note: For all the tests below, I'm emitting event `CurrentTransactionBlockNumber` which provides the following info for reward transaction: blocknumber, timestamp, expirationTimestamp, timeDifferenceRewardTxBlockvsExpirationBlock(timestamp difference between tx timestamp and expiration timestamp)

## Test 1: No changes to current implementation

Conditions:

* `vm.warp(block.timestamp + rewardTimelock.txCooldown() + rewardTimelock.txExpiration() - 0 seconds);`
* `txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration > block.timestamp,` Test result:

```solidity
    │   │   ├─ [5122] TransparentUpgradeableProxy::executeRewardTransaction(0x3871de45d2c1795a66a4197b4a94fe3dbe5c9ef0d11adb60e4a1a39f3fe71aca, 0, [], 1000000000000000000 [1e18], 50000 [5e4])
    │   │   │   ├─ [4284] RewardTimelock::executeRewardTransaction(0x3871de45d2c1795a66a4197b4a94fe3dbe5c9ef0d11adb60e4a1a39f3fe71aca, 0, [], 1000000000000000000 [1e18], 50000 [5e4]) [delegatecall]
    │   │   │   │   ├─ emit CurrentTransactionBlockNumber(blocknumber: 1, timestamp: 950401 [9.504e5], expirationTimestamp: 950401 [9.504e5], timeDifferenceRewardTxBlockvsExpirationBlock: 0)
    │   │   │   │   └─ ← revert: RewardTimelock: transaction is expired
    │   │   │   └─ ← revert: RewardTimelock: transaction is expired
    │   │   └─ ← revert: GS013
    │   └─ ← revert: GS013
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 13.19ms (2.64ms CPU time)

Ran 2 test suites in 1.87s (24.43ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

Summary: `RewardTimelock: transaction is expired` `emit CurrentTransactionBlockNumber(blocknumber: 1, timestamp: 950401 [9.504e5], expirationTimestamp: 950401 [9.504e5], timeDifferenceRewardTxBlockvsExpirationBlock: 0)`

## Test 2: With changes to current implementation: my bugfix implemented: use >= instead of >

Conditions:

* `vm.warp(block.timestamp + rewardTimelock.txCooldown() + rewardTimelock.txExpiration() - 0 seconds);`
* `txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration >= block.timestamp,` Test result:

```solidity
    │   │   ├─ [26528] TransparentUpgradeableProxy::executeRewardTransaction(0x5a693986fb9bb4c3d194b72f4405fbe0efc543d37d756574d22bc552445c0e9d, 0, [], 1000000000000000000 [1e18], 50000 [5e4])
    │   │   │   ├─ [25705] RewardTimelock::executeRewardTransaction(0x5a693986fb9bb4c3d194b72f4405fbe0efc543d37d756574d22bc552445c0e9d, 0, [], 1000000000000000000 [1e18], 50000 [5e4]) [delegatecall]
    │   │   │   │   ├─ emit CurrentTransactionBlockNumber(blocknumber: 1, timestamp: 950401 [9.504e5], expirationTimestamp: 950401 [9.504e5], timeDifferenceRewardTxBlockvsExpirationBlock: 0)
    │   │   │   │   ├─ [1406] TransparentUpgradeableProxy::isFrozen(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   │   │   ├─ [608] VaultFreezer::isFrozen(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   │   │   │   │   └─ ← false
    │   │   │   │   │   └─ ← false
    │   │   │   │   ├─ [0] TransparentUpgradeableProxy::vaultIsInArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   │   │   └─ ← true
    │   │   │   │   ├─ [1444] TransparentUpgradeableProxy::timeSinceOngoingArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   │   │   ├─ [646] Arbitration::timeSinceOngoingArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   │   │   │   │   └─ ← 0
    │   │   │   │   │   └─ ← 0
    │   │   │   │   ├─ emit TransactionExecuted(txHash: 0x5a693986fb9bb4c3d194b72f4405fbe0efc543d37d756574d22bc552445c0e9d, to: RewardTimelockTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], vault: GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2], dollarAmount: 2000, tokenAmounts: [], nativeTokenAmount: 1000000000000000000 [1e18])
    │   │   │   │   ├─ [5750] PriceConsumer::tryGetSaneUsdPrice18Decimals(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE) [staticcall]
    │   │   │   │   │   ├─ [0] feedRegistry::latestRoundData(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE, 0x0000000000000000000000000000000000000348) [staticcall]
    │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   └─ ← EvmError: Revert
    │   │   │   │   └─ ← EvmError: Revert
    │   │   │   └─ ← EvmError: Revert
    │   │   └─ ← revert: GS013
    │   └─ ← revert: GS013
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 12.52ms (3.05ms CPU time)

Ran 2 test suites in 1.95s (23.15ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

Summary: `emit TransactionExecuted` `emit CurrentTransactionBlockNumber(blocknumber: 1, timestamp: 950401 [9.504e5], expirationTimestamp: 950401 [9.504e5], timeDifferenceRewardTxBlockvsExpirationBlock: 0)`

## Test 3: No changes to current implementation but changes to test function's vm.warp method

Conditions:

* `vm.warp(block.timestamp + rewardTimelock.txCooldown() + rewardTimelock.txExpiration() - 1 seconds);`
* `txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration > block.timestamp,` Test result:

```solidity
    │   │   ├─ [26525] TransparentUpgradeableProxy::executeRewardTransaction(0x3871de45d2c1795a66a4197b4a94fe3dbe5c9ef0d11adb60e4a1a39f3fe71aca, 0, [], 1000000000000000000 [1e18], 50000 [5e4])
    │   │   │   ├─ [25702] RewardTimelock::executeRewardTransaction(0x3871de45d2c1795a66a4197b4a94fe3dbe5c9ef0d11adb60e4a1a39f3fe71aca, 0, [], 1000000000000000000 [1e18], 50000 [5e4]) [delegatecall]
    │   │   │   │   ├─ emit CurrentTransactionBlockNumber(blocknumber: 1, timestamp: 950400 [9.504e5], expirationTimestamp: 950401 [9.504e5], timeDifferenceRewardTxBlockvsExpirationBlock: -1)
    │   │   │   │   ├─ [1406] TransparentUpgradeableProxy::isFrozen(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   │   │   ├─ [608] VaultFreezer::isFrozen(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   │   │   │   │   └─ ← false
    │   │   │   │   │   └─ ← false
    │   │   │   │   ├─ [0] TransparentUpgradeableProxy::vaultIsInArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   │   │   └─ ← true
    │   │   │   │   ├─ [1444] TransparentUpgradeableProxy::timeSinceOngoingArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   │   │   ├─ [646] Arbitration::timeSinceOngoingArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   │   │   │   │   └─ ← 0
    │   │   │   │   │   └─ ← 0
    │   │   │   │   ├─ emit TransactionExecuted(txHash: 0x3871de45d2c1795a66a4197b4a94fe3dbe5c9ef0d11adb60e4a1a39f3fe71aca, to: RewardTimelockTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], vault: GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2], dollarAmount: 2000, tokenAmounts: [], nativeTokenAmount: 1000000000000000000 [1e18])
    │   │   │   │   ├─ [5750] PriceConsumer::tryGetSaneUsdPrice18Decimals(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE) [staticcall]
    │   │   │   │   │   ├─ [0] feedRegistry::latestRoundData(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE, 0x0000000000000000000000000000000000000348) [staticcall]
    │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   └─ ← EvmError: Revert
    │   │   │   │   └─ ← EvmError: Revert
    │   │   │   └─ ← EvmError: Revert
    │   │   └─ ← revert: GS013
    │   └─ ← revert: GS013
    └─ ← ()

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 11.55ms (2.36ms CPU time)

Ran 2 test suites in 2.29s (23.07ms CPU time): 1 tests passed, 1 failed, 0 skipped (2 total tests)

Failing tests:
Encountered 1 failing test in test/foundry/RewardTimelock.t.sol:RewardTimelockTest
[FAIL. Reason: assertion failed] testTransactionExpiredRevertsExecution() (gas: 385939)

Encountered a total of 1 failing tests, 1 tests succeeded
```

Summary: `emit TransactionExecuted` `emit CurrentTransactionBlockNumber(blocknumber: 1, timestamp: 950400 [9.504e5], expirationTimestamp: 950401 [9.504e5], timeDifferenceRewardTxBlockvsExpirationBlock: -1)`

* Explanation: with your currently implemented logic the reward tx will revert when tx `block.timestamp` and timestamp of tx expiry overlap, e.g. when `timestamp == expirationTimestamp == 950401`, but if I make the expiry timestamp 1 second later, then we get the above test result, it does not revert.
* With my bugfix it does not revert when `timestamp == expirationTimestamp == 950401`.
* So it's up to Immunefi team's idea of how they want this to work. Do you prefer method A or method B which I described earlier in this report? Either one is OK to use, but consistency is important, as well as clarity. New devs or devs from other protocols, as well as users/whitehats should not be confused or get incorrect ideas of how your protocol logic handles reward tx expiry.
* Why is all this so critically important? Because if any of these depended on critical state changes, it matters whether you use `>` or `>=`, AND it matters that your dev team remembers why `>` is used instead of `>=`. Otherwise invalid state changes could be at risk of happening, although not in this case though.

I will continue hunting for other cases where it might be more serious...

## My recommendation/bugfix:

```diff
require(
-   txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration > block.timestamp,
+   txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration >= block.timestamp,
    "RewardTimelock: transaction is expired"
);
```
