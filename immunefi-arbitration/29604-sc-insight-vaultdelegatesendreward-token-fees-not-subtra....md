# 29604 - \[SC - Insight] VaultDelegatesendReward - Token fees not subtra...

## `VaultDelegate::sendReward()` - Token fees not subtracted from vault balance before rewards are sent, triggering DoS of reward distribution functionality.

Submitted on Mar 27th 2024 at 05:40:27 UTC by @OxSCSamurai for [Boost | Immunefi Arbitration](https://immunefi.com/bounty/immunefiarbitration-boost/)

Report ID: #29604

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/immunefi-team/vaults/blob/main/src/common/VaultDelegate.sol>

Impacts:

* Smart contract unable to operate due to lack of token funds
* Temporary freezing of funds
* Contract fails to deliver promised returns, but doesn't lose value

### Description

### Brief/Intro

(This bug report focuses on the native token section of the affected function, but the bug appears to be present in the ERC20 token section too, in the affected function. So this report + bugfix should apply to the ERC20 section too with the appropriate modifications of course.)

The existing protocol test `RewardTimelock.t.sol::testQueuesAndExecutesRewardTx()` could not/did not catch this bug because of the following reasons:

* total of `1.1 ether` was deposited into the vault via `vm.deal()` cheat-code function, but only `1 ether` was queued during queued transaction for the native token amount, instead of the full `1.1 ether`. And since the native token fee worked out to `0.1` ether exactly, the bug was not triggered, and the reward distribution transaction was successful with zero balance left in the vault afterwards.

### Vulnerability Details

## What's the bug exactly?

So the bug is that the `sendReward()` function does not subtract the fee from the vault balance, and then uses the same vault balance to send the reward to the whitehat. Simple example:

* If the total vault balance is 100, and the fee is 10, then after the fee was sent the reward amount to be sent is still 100, instead of 90.
* But the existing protocol test missed this as per previous explanation above.

My PoC down below demonstrates how the reward is actually successfully sent WITH the bugfix, but the transaction fails WITHOUT the bugfix.

This is where the native token fee amount is calculated: `uint256 nativeTokenFee = (nativeTokenAmount * feeBps) / feeBasis;` And this is where the same `nativeTokenAmount` as above is used to send the reward to the whitehat: `(bool success, ) = to.call{ value: nativeTokenAmount, gas: gasToTarget }("");` The `nativeTokenFee` should have been subtracted from the `nativeTokenAmount` first, but it wasn't.

The buggy function:

```solidity
    function sendReward(
        uint96 referenceId,
        address to,
        Rewards.ERC20Reward[] calldata tokenAmounts,
        uint256 nativeTokenAmount,
        uint256 gasToTarget
    ) external {
        require(gasToTarget <= UNTRUSTED_TARGET_GAS_CAP, "VaultDelegate: gasToTarget greater than max allowed");
        (uint16 feeBps, address feeRecipient) = vaultFees.getFee(address(this));
        uint256 feeBasis = vaultFees.FEE_BASIS();

        // checks on inputs were done when building tx
        emit RewardSent(address(this), referenceId, to, tokenAmounts, nativeTokenAmount, feeRecipient, feeBps);

        uint256 length = tokenAmounts.length;
        for (uint256 i = 0; i < length; i++) {
            if (tokenAmounts[i].amount == 0) {
                continue;
            }
            uint256 tokenFee = (tokenAmounts[i].amount * feeBps) / feeBasis;
            if (tokenFee > 0) {
                require(
                    transferToken(tokenAmounts[i].token, feeRecipient, tokenFee),
                    "VaultDelegate: token transfer to fee recipient failed"
                );
            }
            require(
                transferToken(tokenAmounts[i].token, to, tokenAmounts[i].amount),
                "VaultDelegate: token transfer failed"
            );
        }

        if (nativeTokenAmount == 0) {
            return;
        }

        uint256 nativeTokenFee = (nativeTokenAmount * feeBps) / feeBasis;
        if (nativeTokenFee > 0) {
            // feeRecipient is trusted, we can skip this check
            // slither-disable-next-line arbitrary-send-eth,low-level-calls
            (bool successFee, ) = feeRecipient.call{ value: nativeTokenFee }("");
            require(successFee, "VaultDelegate: Failed to send ether to fee receiver");
        }

        // slither-disable-next-line arbitrary-send-eth,low-level-calls
        (bool success, ) = to.call{ value: nativeTokenAmount, gas: gasToTarget }("");
        require(success, "VaultDelegate: Failed to send native token");
    }
```

### Impact Details

## IMPACT:

Impacts in Scope:

* Smart contract unable to operate due to lack of token funds
* Contract fails to deliver promised returns, but doesn't lose value
* Maybe, depends on your opinion too: Temporary freezing of funds >>> at least for the whitehat, in terms of not getting paid their reward when they expected to, a delay in reward payment due to the bug could potentially constitute as a temporary freezing of funds...?
* if it wasn't for your `sendRewardNoFees()` function and your ability to withdraw both ERC20 & native tokens from the vaults, this bug would most certainly be either a critical severity or at least a high. Therefore I deem the severity at least medium.
* For most/all `sendReward()` function calls the transaction will revert whenever the fee is non-zero, because once the fee has been sent the `nativeTokenAmount` value will be greater than the total funds remaining in the vault, and therefore the transaction will revert at this point:

```solidity
        (bool success, ) = to.call{ value: nativeTokenAmount, gas: gasToTarget }("");
        require(success, "VaultDelegate: Failed to send native token");
```

### References

<https://github.com/immunefi-team/vaults/blob/49c1de26cda19c9e8a4aa311ba3b0dc864f34a25/src/common/VaultDelegate.sol#L61-L114>

### Proof of Concept

## PoC:

Existing protocol test function used, with some modifications:

```solidity
    function testQueuesAndExecutesRewardTx() public {
        uint256 value = 1.1 ether;
        //uint256 dollarAmount = 2000;
        uint256 dollarAmount = 2200; /// @audit added for PoC/testing purposes
        vm.deal(address(vault), value);
        assertEq(address(vault).balance, 1.1 ether); /// @audit added for PoC/testing purposes

        // set right permissions on moduleGuard
        vm.startPrank(protocolOwner);
        moduleGuard.setTargetAllowed(address(vaultDelegate), true);
        moduleGuard.setAllowedFunction(address(vaultDelegate), vaultDelegate.sendReward.selector, true);
        moduleGuard.setDelegateCallAllowedOnTarget(address(vaultDelegate), true);
        vm.stopPrank();

        uint256 nonce = rewardTimelock.vaultTxNonce(address(vault));
        bytes32 txHash = rewardTimelock.getQueueTransactionHash(address(this), dollarAmount, address(vault), nonce);

        // Mock vaultIsInArbitration
        vm.mockCall(
            address(arbitration),
            abi.encodeCall(arbitration.vaultIsInArbitration, (address(vault))),
            abi.encode(true)
        );

        vm.expectEmit(true, true, true, true);
        emit TransactionQueued(txHash, address(this), address(vault), dollarAmount);
        _sendTxToVault(
            address(rewardTimelock),
            0,
            abi.encodeCall(rewardTimelock.queueRewardTransaction, (address(this), dollarAmount)),
            Enum.Operation.Call
        );

        assertEq(rewardTimelock.vaultTxNonce(address(vault)), nonce + 1);

        vm.warp(block.timestamp + 1 hours);
        assertFalse(rewardTimelock.canExecuteTransaction(txHash));

        vm.warp(block.timestamp + rewardTimelock.txCooldown() - 1 hours);
        //vm.warp(block.timestamp + rewardTimelock.txCooldown() + 100 hours); /// @audit added for PoC/testing purposes
        assertTrue(rewardTimelock.canExecuteTransaction(txHash));

        Rewards.ERC20Reward[] memory erc20Rewards = new Rewards.ERC20Reward[](0);

        // Mock priceConsumer
        vm.mockCall(
            address(priceConsumer),
            abi.encodeCall(priceConsumer.tryGetSaneUsdPrice18Decimals, (Denominations.ETH)),
            abi.encode(uint256(2000) * 10 ** 18)
        );

        vm.expectEmit(true, true, true, true);
        emit TransactionExecuted(txHash, address(this), address(vault), dollarAmount, erc20Rewards, 1 ether);
        _sendTxToVault(
            address(rewardTimelock),
            0,
            //abi.encodeCall(rewardTimelock.executeRewardTransaction, (txHash, 0, erc20Rewards, 1 ether, 50_000)),
            //abi.encodeCall(rewardTimelock.executeRewardTransaction, (txHash, 0, erc20Rewards, 1.01 ether, 50_000)), /// @audit added for PoC/testing purposes
            abi.encodeCall(rewardTimelock.executeRewardTransaction, (txHash, 0, erc20Rewards, 1.1 ether, 50_000)), /// @audit added for PoC/testing purposes
            Enum.Operation.Call
        );
        assertEq(rewardTimelock.vaultTxNonce(address(vault)), nonce + 1);
        assertEq(address(vault).balance, 0);
    }
```

The PoC tests: (make sure to check what specific cases I used in above test function). For example, in the default protocol test, it sends `1 ether` as parameter value for native token value for queue reward tx and also for execute reward tx, but with my tests below I use `1.1 ether` for the parameter value, which is the correct representation of the vault balance, not `1 ether`.

Forge command used: `forge test --contracts test/foundry/RewardTimelock.t.sol --mt testQueuesAndExecutesRewardTx -vvvvv`

## Test 1: WITHOUT bugfix:

```solidity
    │   │   │   │   │   │   │   │   ├─ [54743] VaultDelegate::sendReward(0, RewardTimelockTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], [], 1100000000000000000 [1.1e18], 50000 [5e4]) [delegatecall]
    │   │   │   │   │   │   │   │   │   ├─ [5044] VaultFees::getFee(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   │   │   │   │   │   │   │   └─ ← 1000, feeRecipient: [0xa32c0203D5F9Fcfe1b2A359e5628cb5a22001bA7]
    │   │   │   │   │   │   │   │   │   ├─ [306] VaultFees::FEE_BASIS() [staticcall]
    │   │   │   │   │   │   │   │   │   │   └─ ← 10000 [1e4]
    │   │   │   │   │   │   │   │   │   ├─ emit RewardSent(vault: GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2], referenceId: 0, to: RewardTimelockTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], tokenAmounts: [], nativeTokenAmount: 1100000000000000000 [1.1e18], feeRecipient: feeRecipient: [0xa32c0203D5F9Fcfe1b2A359e5628cb5a22001bA7], fee: 1000)
    │   │   │   │   │   │   │   │   │   ├─ [0] feeRecipient::fallback{value: 110000000000000000}()
    │   │   │   │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   │   │   │   ├─ [0] RewardTimelockTest::receive{value: 1100000000000000000}()
    │   │   │   │   │   │   │   │   │   │   └─ ← EvmError: OutOfFunds
    │   │   │   │   │   │   │   │   │   └─ ← revert: VaultDelegate: Failed to send native token
    │   │   │   │   │   │   │   │   ├─ emit ExecutionFromModuleFailure(module: TransparentUpgradeableProxy: [0xd42300156aa9Ee8b8B3B0DbB2bd82416163De7Cc])
    │   │   │   │   │   │   │   │   └─ ← false
    │   │   │   │   │   │   │   └─ ← false
    │   │   │   │   │   │   ├─ [1217] TransparentUpgradeableProxy::checkAfterExecution(0x3078000000000000000000000000000000000000000000000000000000000000, false)
    │   │   │   │   │   │   │   ├─ [419] ScopeGuard::checkAfterExecution(0x3078000000000000000000000000000000000000000000000000000000000000, false) [delegatecall]
    │   │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   └─ ← revert: ImmunefiModule: execution failed
    │   │   │   │   │   └─ ← revert: ImmunefiModule: execution failed
    │   │   │   │   └─ ← revert: ImmunefiModule: execution failed
    │   │   │   └─ ← revert: ImmunefiModule: execution failed
    │   │   └─ ← revert: GS013
    │   └─ ← revert: GS013
    └─ ← revert: GS013

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 11.80ms (2.57ms CPU time)

Ran 1 test suite in 1.91s (11.80ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/foundry/RewardTimelock.t.sol:RewardTimelockTest
[FAIL. Reason: revert: GS013] testQueuesAndExecutesRewardTx() (gas: 469429)

Encountered a total of 1 failing tests, 0 tests succeeded
```

Test result: FAILED. Reward NOT sent to whitehat:

```solidity
    │   │   │   │   │   │   │   │   │   ├─ [0] RewardTimelockTest::receive{value: 1100000000000000000}()
    │   │   │   │   │   │   │   │   │   │   └─ ← EvmError: OutOfFunds
    │   │   │   │   │   │   │   │   │   └─ ← revert: VaultDelegate: Failed to send native token
```

## Test 2: WITH bugfix:

```solidity
    │   │   │   │   │   │   │   │   ├─ [54754] VaultDelegate::sendReward(0, RewardTimelockTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], [], 1100000000000000000 [1.1e18], 50000 [5e4]) [delegatecall]
    │   │   │   │   │   │   │   │   │   ├─ [5044] VaultFees::getFee(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   │   │   │   │   │   │   │   └─ ← 1000, feeRecipient: [0xa32c0203D5F9Fcfe1b2A359e5628cb5a22001bA7]
    │   │   │   │   │   │   │   │   │   ├─ [306] VaultFees::FEE_BASIS() [staticcall]
    │   │   │   │   │   │   │   │   │   │   └─ ← 10000 [1e4]
    │   │   │   │   │   │   │   │   │   ├─ emit RewardSent(vault: GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2], referenceId: 0, to: RewardTimelockTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], tokenAmounts: [], nativeTokenAmount: 1100000000000000000 [1.1e18], feeRecipient: feeRecipient: [0xa32c0203D5F9Fcfe1b2A359e5628cb5a22001bA7], fee: 1000)
    │   │   │   │   │   │   │   │   │   ├─ [0] feeRecipient::fallback{value: 110000000000000000}()
    │   │   │   │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   │   │   │   ├─ [55] RewardTimelockTest::receive{value: 990000000000000000}()
    │   │   │   │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   │   │   ├─ emit ExecutionFromModuleSuccess(module: TransparentUpgradeableProxy: [0xd42300156aa9Ee8b8B3B0DbB2bd82416163De7Cc])
    │   │   │   │   │   │   │   │   └─ ← true
    │   │   │   │   │   │   │   └─ ← true
    │   │   │   │   │   │   ├─ [1217] TransparentUpgradeableProxy::checkAfterExecution(0x3078000000000000000000000000000000000000000000000000000000000000, true)
    │   │   │   │   │   │   │   ├─ [419] ScopeGuard::checkAfterExecution(0x3078000000000000000000000000000000000000000000000000000000000000, true) [delegatecall]
    │   │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   └─ ← ()
    │   │   │   │   └─ ← ()
    │   │   │   └─ ← ()
    │   │   ├─ emit ExecutionSuccess(txHash: 0xb02c9832b6b599b2da46eb47716a2afb30a542a718f426c2ca9ae05f21bbf709, payment: 0)
    │   │   ├─ [1127] TransparentUpgradeableProxy::checkAfterExecution(0xb02c9832b6b599b2da46eb47716a2afb30a542a718f426c2ca9ae05f21bbf709, true)
    │   │   │   ├─ [329] ImmunefiGuard::checkAfterExecution(0xb02c9832b6b599b2da46eb47716a2afb30a542a718f426c2ca9ae05f21bbf709, true) [delegatecall]
    │   │   │   │   └─ ← ()
    │   │   │   └─ ← ()
    │   │   └─ ← true
    │   └─ ← true
    ├─ [1362] TransparentUpgradeableProxy::vaultTxNonce(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   ├─ [564] RewardTimelock::vaultTxNonce(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   └─ ← 1
    │   └─ ← 1
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 12.89ms (2.76ms CPU time)

Ran 1 test suite in 2.77s (12.89ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Test result: SUCCESS. Successfully sent reward to whitehat: `ImmunefiGuard::checkAfterExecution(0xb02c9832b6b599b2da46eb47716a2afb30a542a718f426c2ca9ae05f21bbf709, true)`

## Test 3: default protocol test without any modifications and without any bugfix applied:

* of course here the test passes because it cheat-code sends `1.1 ether` into vault, and then it queues and executes reward transaction for only `1 ether` instead of `1.1 ether`, thereby allowing the bug to be overlooked because the fee is only `0.1 ether`, so it works out perfectly to bypass the bug, which my tests above have proven to exist:

```solidity
    │   │   │   │   │   │   │   │   ├─ [54688] VaultDelegate::sendReward(0, RewardTimelockTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], [], 1000000000000000000 [1e18], 50000 [5e4]) [delegatecall]
    │   │   │   │   │   │   │   │   │   ├─ [5044] VaultFees::getFee(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   │   │   │   │   │   │   │   │   │   └─ ← 1000, feeRecipient: [0xa32c0203D5F9Fcfe1b2A359e5628cb5a22001bA7]
    │   │   │   │   │   │   │   │   │   ├─ [306] VaultFees::FEE_BASIS() [staticcall]
    │   │   │   │   │   │   │   │   │   │   └─ ← 10000 [1e4]
    │   │   │   │   │   │   │   │   │   ├─ emit RewardSent(vault: GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2], referenceId: 0, to: RewardTimelockTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], tokenAmounts: [], nativeTokenAmount: 1000000000000000000 [1e18], feeRecipient: feeRecipient: [0xa32c0203D5F9Fcfe1b2A359e5628cb5a22001bA7], fee: 1000)
    │   │   │   │   │   │   │   │   │   ├─ [0] feeRecipient::fallback{value: 100000000000000000}()
    │   │   │   │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   │   │   │   ├─ [55] RewardTimelockTest::receive{value: 1000000000000000000}()
    │   │   │   │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   │   │   ├─ emit ExecutionFromModuleSuccess(module: TransparentUpgradeableProxy: [0xd42300156aa9Ee8b8B3B0DbB2bd82416163De7Cc])
    │   │   │   │   │   │   │   │   └─ ← true
    │   │   │   │   │   │   │   └─ ← true
    │   │   │   │   │   │   ├─ [1217] TransparentUpgradeableProxy::checkAfterExecution(0x3078000000000000000000000000000000000000000000000000000000000000, true)
    │   │   │   │   │   │   │   ├─ [419] ScopeGuard::checkAfterExecution(0x3078000000000000000000000000000000000000000000000000000000000000, true) [delegatecall]
    │   │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   │   └─ ← ()
    │   │   │   │   │   └─ ← ()
    │   │   │   │   └─ ← ()
    │   │   │   └─ ← ()
    │   │   ├─ emit ExecutionSuccess(txHash: 0x8efa05a37ab747cfedb358632ee8acb272e16f16217125c1a952549c29bc244c, payment: 0)
    │   │   ├─ [1127] TransparentUpgradeableProxy::checkAfterExecution(0x8efa05a37ab747cfedb358632ee8acb272e16f16217125c1a952549c29bc244c, true)
    │   │   │   ├─ [329] ImmunefiGuard::checkAfterExecution(0x8efa05a37ab747cfedb358632ee8acb272e16f16217125c1a952549c29bc244c, true) [delegatecall]
    │   │   │   │   └─ ← ()
    │   │   │   └─ ← ()
    │   │   └─ ← true
    │   └─ ← true
    ├─ [1362] TransparentUpgradeableProxy::vaultTxNonce(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
    │   ├─ [564] RewardTimelock::vaultTxNonce(GnosisSafeProxy: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [delegatecall]
    │   │   └─ ← 1
    │   └─ ← 1
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.76ms (2.54ms CPU time)

Ran 1 test suite in 2.05s (11.76ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Test result: pass. No bugfix here but it passes due to the explanation above.

* Lesson here: always double check your protocol unit/integration tests and make sure you cover all possible cases, including edge cases.
* Also, your `sendRewardNoFees()` function has exactly the same code implementation except without the fee logic. This function works correctly because it doesnt have fees to account for.

## BUGFIX SUGGESTION:

```diff
    function sendReward(
        uint96 referenceId,
        address to,
        Rewards.ERC20Reward[] calldata tokenAmounts,
        uint256 nativeTokenAmount,
        uint256 gasToTarget
    ) external {
        require(gasToTarget <= UNTRUSTED_TARGET_GAS_CAP, "VaultDelegate: gasToTarget greater than max allowed");
        (uint16 feeBps, address feeRecipient) = vaultFees.getFee(address(this));
        uint256 feeBasis = vaultFees.FEE_BASIS();

        // checks on inputs were done when building tx
        emit RewardSent(address(this), referenceId, to, tokenAmounts, nativeTokenAmount, feeRecipient, feeBps);

        uint256 length = tokenAmounts.length;
        for (uint256 i = 0; i < length; i++) {
            if (tokenAmounts[i].amount == 0) {
                continue;
            }
            uint256 tokenFee = (tokenAmounts[i].amount * feeBps) / feeBasis;
            if (tokenFee > 0) {
                require(
                    transferToken(tokenAmounts[i].token, feeRecipient, tokenFee),
                    "VaultDelegate: token transfer to fee recipient failed"
                );
            }
            require(
-               transferToken(tokenAmounts[i].token, to, tokenAmounts[i].amount),
+               transferToken(tokenAmounts[i].token, to, (tokenAmounts[i].amount - tokenFee)),
                "VaultDelegate: token transfer failed"
            );
        }

        if (nativeTokenAmount == 0) {
            return;
        }

        uint256 nativeTokenFee = (nativeTokenAmount * feeBps) / feeBasis;
        if (nativeTokenFee > 0) {
            // feeRecipient is trusted, we can skip this check
            // slither-disable-next-line arbitrary-send-eth,low-level-calls
            (bool successFee, ) = feeRecipient.call{ value: nativeTokenFee }("");
            require(successFee, "VaultDelegate: Failed to send ether to fee receiver");
        }

        // slither-disable-next-line arbitrary-send-eth,low-level-calls
-       (bool success, ) = to.call{ value: nativeTokenAmount, gas: gasToTarget }("");
+       (bool success, ) = to.call{ value: (nativeTokenAmount - nativeTokenFee), gas: gasToTarget }("");
        require(success, "VaultDelegate: Failed to send native token");
    }
```
