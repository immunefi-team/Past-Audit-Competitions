# 29483 - \[SC - Insight] RewardTimelockcanExecuteTransaction - Reward tr...

## `RewardTimelock::canExecuteTransaction()` - Reward transaction allowed to execute before the cooldown period has strictly passed.

Submitted on Mar 20th 2024 at 21:59:38 UTC by @OxSCSamurai for [Boost | Immunefi Arbitration](https://immunefi.com/bounty/immunefiarbitration-boost/)

Report ID: #29483

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/immunefi-team/vaults/blob/main/src/RewardTimelock.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

### Description

### Brief/Intro

Note: The primary reason for this report is to draw your attention to something important, which if neglected, could in some cases/scenarios result in a high/critical vulnerability. In other words, anything other than PRECISE use of comparison operators will inevitably eventually result in a serious vulnerability, somewhere along the line... This report attempts to make it crystal clear why the difference between an inequality operator with an `=` sign and without an `=` sign is significant, and ALWAYS matters.

### Vulnerability Details

The "buggy" function:

```solidity
    function canExecuteTransaction(bytes32 txHash) external view returns (bool) {
        TxStorageData memory txData = txHashData[txHash];

        if (vaultFreezer.isFrozen(txData.vault)) return false;
        if (!arbitration.vaultIsInArbitration(txData.vault)) return false;
        // @dev Time shouldn't be 0 if vault is in arbitration.
        // @dev At least txCooldown time should have passed since the vault entered into an arbitration.
        if (block.timestamp - arbitration.timeSinceOngoingArbitration(txData.vault) < txData.cooldown) return false;

        return
            txData.state == TxState.Queued &&
            txData.queueTimestamp + txData.cooldown <= block.timestamp &&
            //txData.queueTimestamp + txData.cooldown < block.timestamp && /// @audit added for PoC/testing purposes
            (txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration > block.timestamp);
    }
```

Given the requirement that the cooldown period must have elapsed before the transaction can be executed, the correct expression should ensure that the current timestamp is strictly greater than the timestamp when the reward transaction was queued plus the cooldown period.

Thus, the correct expression is: `txData.queueTimestamp + txData.cooldown < block.timestamp` not: `txData.queueTimestamp + txData.cooldown <= block.timestamp`

So, in the case(not necessarily relevant here but could have been) where a state change needs to happen AT exactly `queuedRewardTxTimestamp + rewardTimelock.txCooldown()` but BEFORE reward transaction's `block.timestamp`, then using `txData.queueTimestamp + txData.cooldown < block.timestamp` instead of `<=` ensures that the state change occurs at the exact moment when the cooldown period ends but before the current reward tx gets executed, at least 1 second apart but still within the same block. (Unless we explicitly separate these transactions by at least 12 seconds, i.e. the average block time.)

If we were to use `<=` instead, this would open up the possibility of executing the reward tx BEFORE the relevant state change gets executed. If arguments against my argument insist that this 1 second difference is negligible, then I could ping pong back an argument that states that OK, if you insist the difference between `<` and `<=` is negligible in this specific case, then I can just as easily insist that you should rather use `<` because even though the difference, in your opinion, is negligible, it does however ensure that the state change can never happen after the reward tx, besides the fact that using `<` would be more in line with the language/grammar, although this is of secondary importance. Getting the actual implemented logic 100% right is what really matters.

## Awesome Analogy:

* Imagine you are on a mission to infiltrate a high-security facility, and to reach the other side, you must cross a narrow bridge that spans a treacherous chasm. At the end of the bridge, there is a deadly vaporizing laser that will instantly vaporize anyone who lingers within its range. The secret instructions for the mission are clear: you must start at one end of the bridge and stop only after PASSING beyond the end of the bridge. If you stop prematurely, the vaporizing laser will detect you and vaporize you, but if you follow the instructions and go beyond the end of the bridge, you will be out of harm's way and can continue your mission safely.
* In this analogy, the 100m strip of bridge represents the queued transaction's delay period, the deadly vaporizing laser represents the danger zone, and the instructions to stop only after passing beyond the end of the bridge are crucial to avoid getting vaporized.
* Similarly, in the context of the cooldown period and the queued transaction, the correct logic for allowing the execution of the queued transaction after the cooldown period is fully completed would be: `txData.queueTimestamp + txData.cooldown < block.timestamp`.
* This ensures that the entire cooldown period has passed since the transaction was queued, and only then can the transaction be executed.

## So to summarize:

* `txData.queueTimestamp + txData.cooldown <= block.timestamp`: This condition dictates that the transaction can be executed precisely when the cooldown period ends or at the same time, potentially overlapping with the danger zone of the vaporizing laser. It's akin to stopping your advance at the very edge of the bridge, where even the slightest overlap risks encountering the deadly laser, disregarding the imperative to surpass the end of the bridge for safety.
* `txData.queueTimestamp + txData.cooldown < block.timestamp`: This alternative condition specifies that the transaction execution must occur strictly after the cooldown period has concluded. Much like the necessity of moving beyond the bridge's end to avoid the vaporizing laser, this logic guarantees that the transaction executes precisely when it's safe to do so.

### Impact Details

## IMPACT:

* Likelihood: High, Impact: Low = Severity: LOW.
* Potential Impact in Scope: Contract fails to deliver promised returns, but doesn't lose value -- Allowing reward tx to get executed before the cooldown period has FULLY passed.

### References

Add any relevant links to documentation or code

### Proof of Concept

## PoC:

* I've put in some effort to demonstrate the different behaviours of the function's checks down to mere seconds differences in `block.timestamp` of reward tx execution. To this end for each test there is a console2.log emitted so we can see whether the reward tx is allowed to be executed during that `block.timestamp` or not.
* cooldown period seems to be exactly 24hrs according to the protocol tests.

Test function used: `RewardTimelock.t.sol::testTransactionInCooldownRevertsExecution()` Modifications to test function:

```solidity
        //vm.warp(block.timestamp + 1 hours);
        vm.warp(block.timestamp + 23 hours + 3602 seconds); /// @audit added for PoC/testing purposes
        bool canExecuteTransactionTrueOrFalse = rewardTimelock.canExecuteTransaction(txHash); /// @audit added for PoC/testing purposes
        //assertFalse(rewardTimelock.canExecuteTransaction(txHash));
        assertFalse(canExecuteTransactionTrueOrFalse); /// @audit added for PoC/testing purposes
        console2.log("CAN EXECUTE REWARD TX: true/false? ", canExecuteTransactionTrueOrFalse); /// @audit added for PoC/testing purposes

        // Rewards.ERC20Reward[] memory erc20Rewards = new Rewards.ERC20Reward[](0);

        // _sendTxToVault(
        //     address(rewardTimelock),
        //     0,
        //     abi.encodeCall(rewardTimelock.executeRewardTransaction, (txHash, 0, erc20Rewards, 1 ether, 50_000)),
        //     Enum.Operation.Call,
        //     true
        // );
```

## TESTS:

## Test 1: demonstrating the bug, allowing for reward tx execution during cooldown period, instead of only AFTER cooldown ends

Using:

* `txData.queueTimestamp + txData.cooldown <= block.timestamp`
* `vm.warp(block.timestamp + 23 hours + 3600 seconds)` >>> this precise timestamp is technically still part of cooldown
* We expect the test to pass allowing for reward tx execution not AFTER cooldown period passed, but before/at exact boundary end of cooldown period. Test result:

```solidity
    ├─ [0] VM::warp(86401 [8.64e4])
    │   └─ ← ()
    ├─ [15613] TransparentUpgradeableProxy::canExecuteTransaction(0x01a29a5d0509d08e749ae78a2ed2b4f4d493ce63541e945f53995c4a2018758e) [staticcall]
    │   ├─ [14815] RewardTimelock::canExecuteTransaction(0x01a29a5d0509d08e749ae78a2ed2b4f4d493ce63541e945f53995c4a2018758e) [delegatecall]
    │   │   ├─ [1406] TransparentUpgradeableProxy::isFrozen(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   ├─ [608] VaultFreezer::isFrozen(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   │   │   └─ ← false
    │   │   │   └─ ← false
    │   │   ├─ [0] TransparentUpgradeableProxy::vaultIsInArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   └─ ← true
    │   │   ├─ [9944] TransparentUpgradeableProxy::timeSinceOngoingArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   ├─ [2646] Arbitration::timeSinceOngoingArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   │   │   └─ ← 0
    │   │   │   └─ ← 0
    │   │   └─ ← true
    │   └─ ← true
    ├─ emit log(val: "Error: Assertion Failed")
    ├─ [0] VM::store(VM: [0x7109709ECfa91a80626fF3989D68f67F5b1DD12D], 0x6661696c65640000000000000000000000000000000000000000000000000000, 0x0000000000000000000000000000000000000000000000000000000000000001)
    │   └─ ← ()
    ├─ [0] console::log("CAN EXECUTE REWARD TX: true/false? ", true) [staticcall]
    │   └─ ← ()
    └─ ← ()

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 11.10ms (1.42ms CPU time)
```

Ignore the failed here and check my console2.log result only: `console::log("CAN EXECUTE REWARD TX: true/false? ", true)`

## Test 2: demonstrating the bugfix, allowing for reward tx execution only AFTER cooldown period.

Using:

* `txData.queueTimestamp + txData.cooldown < block.timestamp`
* `vm.warp(block.timestamp + 23 hours + 3600 seconds)` >>> this precise timestamp is technically still part of cooldown
* We expect the test to fail preventing reward tx execution before cooldown period has fully passed Test result:

```solidity
    ├─ [0] VM::warp(86401 [8.64e4])
    │   └─ ← ()
    ├─ [15320] TransparentUpgradeableProxy::canExecuteTransaction(0x1f1356a67b9e404855e103f18450698e4b558e4814e7618c574924604d68d6d1) [staticcall]
    │   ├─ [14522] RewardTimelock::canExecuteTransaction(0x1f1356a67b9e404855e103f18450698e4b558e4814e7618c574924604d68d6d1) [delegatecall]
    │   │   ├─ [1406] TransparentUpgradeableProxy::isFrozen(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   ├─ [608] VaultFreezer::isFrozen(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   │   │   └─ ← false
    │   │   │   └─ ← false
    │   │   ├─ [0] TransparentUpgradeableProxy::vaultIsInArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   └─ ← true
    │   │   ├─ [9944] TransparentUpgradeableProxy::timeSinceOngoingArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   ├─ [2646] Arbitration::timeSinceOngoingArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   │   │   └─ ← 0
    │   │   │   └─ ← 0
    │   │   └─ ← false
    │   └─ ← false
    ├─ [0] console::log("CAN EXECUTE REWARD TX: true/false? ", false) [staticcall]
    │   └─ ← ()
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 17.41ms (2.36ms CPU time)
```

Ignore the passed here and check my console2.log result only: `console::log("CAN EXECUTE REWARD TX: true/false? ", false)`

## For next two tests we increase the warp time by 1 second, i.e. 24 hours + 1 seconds.

## Test 3: same as Test 1(with bug) but reward tx executed 1 second later:

Using:

* `txData.queueTimestamp + txData.cooldown <= block.timestamp`
* `vm.warp(block.timestamp + 23 hours + 3601 seconds)` >>> this precise timestamp is technically still part of cooldown
* We expect the test to pass allowing for reward tx execution AFTER cooldown period has passed Test result:

```solidity
    ├─ [0] VM::warp(86402 [8.64e4])
    │   └─ ← ()
    ├─ [15613] TransparentUpgradeableProxy::canExecuteTransaction(0x01a29a5d0509d08e749ae78a2ed2b4f4d493ce63541e945f53995c4a2018758e) [staticcall]
    │   ├─ [14815] RewardTimelock::canExecuteTransaction(0x01a29a5d0509d08e749ae78a2ed2b4f4d493ce63541e945f53995c4a2018758e) [delegatecall]
    │   │   ├─ [1406] TransparentUpgradeableProxy::isFrozen(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   ├─ [608] VaultFreezer::isFrozen(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   │   │   └─ ← false
    │   │   │   └─ ← false
    │   │   ├─ [0] TransparentUpgradeableProxy::vaultIsInArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   └─ ← true
    │   │   ├─ [9944] TransparentUpgradeableProxy::timeSinceOngoingArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   ├─ [2646] Arbitration::timeSinceOngoingArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   │   │   └─ ← 0
    │   │   │   └─ ← 0
    │   │   └─ ← true
    │   └─ ← true
    ├─ emit log(val: "Error: Assertion Failed")
    ├─ [0] VM::store(VM: [0x7109709ECfa91a80626fF3989D68f67F5b1DD12D], 0x6661696c65640000000000000000000000000000000000000000000000000000, 0x0000000000000000000000000000000000000000000000000000000000000001)
    │   └─ ← ()
    ├─ [0] console::log("CAN EXECUTE REWARD TX: true/false? ", true) [staticcall]
    │   └─ ← ()
    └─ ← ()

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 12.38ms (1.47ms CPU time)
```

Ignore the failed here and check my console2.log result only: `console::log("CAN EXECUTE REWARD TX: true/false? ", true)`

## Test 4: demonstrating the bugfix, allowing for reward tx execution only AFTER cooldown period.

Using:

* `txData.queueTimestamp + txData.cooldown < block.timestamp`
* `vm.warp(block.timestamp + 23 hours + 3601 seconds)` >>> this precise timestamp is technically still part of cooldown
* We expect the test to pass allowing reward tx execution strictly AFTER cooldown period has fully passed Test result:

```solidity
    ├─ [0] VM::warp(86402 [8.64e4])
    │   └─ ← ()
    ├─ [15610] TransparentUpgradeableProxy::canExecuteTransaction(0x1f1356a67b9e404855e103f18450698e4b558e4814e7618c574924604d68d6d1) [staticcall]
    │   ├─ [14812] RewardTimelock::canExecuteTransaction(0x1f1356a67b9e404855e103f18450698e4b558e4814e7618c574924604d68d6d1) [delegatecall]
    │   │   ├─ [1406] TransparentUpgradeableProxy::isFrozen(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   ├─ [608] VaultFreezer::isFrozen(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   │   │   └─ ← false
    │   │   │   └─ ← false
    │   │   ├─ [0] TransparentUpgradeableProxy::vaultIsInArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   └─ ← true
    │   │   ├─ [9944] TransparentUpgradeableProxy::timeSinceOngoingArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   ├─ [2646] Arbitration::timeSinceOngoingArbitration(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   │   │   └─ ← 0
    │   │   │   └─ ← 0
    │   │   └─ ← true
    │   └─ ← true
    ├─ emit log(val: "Error: Assertion Failed")
    ├─ [0] VM::store(VM: [0x7109709ECfa91a80626fF3989D68f67F5b1DD12D], 0x6661696c65640000000000000000000000000000000000000000000000000000, 0x0000000000000000000000000000000000000000000000000000000000000001)
    │   └─ ← ()
    ├─ [0] console::log("CAN EXECUTE REWARD TX: true/false? ", true) [staticcall]
    │   └─ ← ()
    └─ ← ()

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 284.36ms (1.36ms CPU time)
```

Ignore the failed here and check my console2.log result only: `console::log("CAN EXECUTE REWARD TX: true/false? ", true)`

## Recommended bugfix:

* Unless there's a valid logical argument against the recommended change below, the change should be implemented. In other words, unless there's a super valid reason why the reward tx can/should be allowed to get executed at EXACTLY `txData.queueTimestamp + txData.cooldown` timestamp, the below bugfix should be implemented.

<https://github.com/immunefi-team/vaults/blob/49c1de26cda19c9e8a4aa311ba3b0dc864f34a25/src/RewardTimelock.sol#L193-L197>

```diff
        return
            txData.state == TxState.Queued &&
-           txData.queueTimestamp + txData.cooldown <= block.timestamp &&
+           txData.queueTimestamp + txData.cooldown < block.timestamp &&
            (txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration > block.timestamp);
```
