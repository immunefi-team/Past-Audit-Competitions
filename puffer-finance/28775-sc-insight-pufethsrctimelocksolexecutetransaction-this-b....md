# 28775 - \[SC - Insight] pufETHsrcTimelocksolexecuteTransaction - This b...

## `pufETH/src/Timelock.sol::executeTransaction()` - This bug makes it possible to unexpectedly execute a timelocked queued transaction TWICE, either accidentally, or intentionally.

Submitted on Feb 26th 2024 at 20:13:12 UTC by @OxSCSamurai for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28775

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Unexpected double execution of transactions queued by operations multisig

### Description

### Brief/Intro

## Summary:

Please read the following carefully:

* This bug makes it possible to unexpectedly execute a timelocked queued transaction TWICE
* either accidentally, involving both operations & community multisigs
* or intentionally involving trusted actors/addresses in either multisigs that went rogue.
* I try to make it clear that the risk here is dependent on EITHER accidental actions or intentional malicious actions.
* I do not imply in any shape or form whatsoever that it requires access to privileged addresses/multisigs. The risk posed by this bug does NOT depend on attackers gaining access to privileged addresses. Please ditch that idea.
* Primary: Due to this bug, ACCIDENTAL actions can lead to unfavorable/unacceptable impacts/risks on the protocol or users.
* Secondary: Due to this bug, malicious actions can lead to unfavorable/unacceptable impacts/risks on the protocol or users.

### Vulnerability Details

* Case 1: operations multisig queues transaction, later executes tx and then community multisig executes same tx again.
* Case 2: operations multisig queues transaction, community multisig executes tx before timelock delay passes, and then operations multisig executes same tx again after timelock delay.
* The bug is related to the lack of proper validation/checks for `queue[txHash] = 0;`.
* For more details and clarity please see the bugfix section for my fixes.
* The bug allows for queued transactions to be executed more than once, which should not be allowed due to the potential risks/impacts involved.
* Could potentially cause damage but at least frustration to protocol and/or users if the same transaction was executed twice. By "same transaction" I'm specifically referring to tx with same `operationId` or `txHash`.
* The invariant being violated by this bug is that `it should never be possible to execute the same queued transaction more than once`.

Type of queued transactions that we dont want to execute twice, i.e. same tx with same operationId executed TWICE:

* upgrade a contract
* depositing into EigenLayer's stETH strategy contract
* redeeming stETH for ETH from EL (via Lido)
* pretty much all the transactions that can be queued by operations multisig now and in the future after upgrades

## The buggy function:

* see my bugfix section after the PoC section:

```solidity
    function executeTransaction(address target, bytes calldata callData, uint256 operationId)
        external
        returns (bool success, bytes memory returnData)
    {
        // Community Multisig can do things without any delay
        if (msg.sender == COMMUNITY_MULTISIG) {
            return _executeTransaction(target, callData);
        }

        // Operations multisig needs to queue it and then execute after a delay
        if (msg.sender != OPERATIONS_MULTISIG) {
            revert Unauthorized();
        }

        bytes32 txHash = keccak256(abi.encode(target, callData, operationId));
        uint256 lockedUntil = queue[txHash];

        // slither-disable-next-line incorrect-equality
        if (lockedUntil == 0) {
            revert InvalidTransaction(txHash);
        }

        if (block.timestamp < lockedUntil) {
        //if (block.timestamp <= lockedUntil) {
        //if (block.timestamp == lockedUntil) {
            revert Locked(txHash, lockedUntil);
        }

        queue[txHash] = 0;
        (success, returnData) = _executeTransaction(target, callData);

        emit TransactionExecuted(txHash, target, callData, operationId);

        return (success, returnData);
    }
```

### Impact Details

## Potential Impacts in scope:

* 1. Griefing
* 2. Contract fails to deliver promised returns, but doesn't lose value

1. Griefing risk is not necessarily isolated to an attacker, it could happen without an attacker/attack, as explained below:

Griefing can encompass situations where disruption, harm, or inconvenience is caused not only by intentional attacks but also by accidental actions or negligence. While griefing often implies malicious intent, the consequences of actions that result from accidental mistakes or negligence can have similar effects on the protocol and its users.

For example:

* Accidental Actions: Suppose a user unintentionally triggers a series of transactions that disrupt the functioning of a decentralized application (DApp) or smart contract, causing financial losses or inconvenience to other users. Even though the action was not malicious, it still results in griefing-like consequences.
* Negligence: If developers or administrators of a protocol fail to properly secure or maintain the system, leading to vulnerabilities being exploited or critical errors occurring, the resulting disruptions and losses could be considered a form of griefing, albeit stemming from negligence rather than malicious intent.

In both cases, whether the disruption is caused intentionally or unintentionally, the impact on the protocol and its users can be similar. It may result in economic losses, loss of trust, reputational damage, and the need for mitigation efforts. Therefore, when discussing griefing in the context of Web3 protocols, it's important to consider a broad spectrum of actions and their consequences, including those stemming from accidents or negligence.

2. The impact "Contract fails to deliver promised returns, but doesn't lose value" can be considered as the contract not behaving as expected, irrespective of whether user funds are involved. In the context of smart contracts, the expected behavior is typically defined by the contract's code and its intended purpose. If the contract fails to fulfill its obligations or execute its functions as intended, it's considered a failure, regardless of whether user funds are directly impacted.

Even if no user funds are involved, such a failure can still have significant repercussions, including loss of trust, reputational damage, legal implications, and economic consequences. Therefore, ensuring that smart contracts behave as expected is crucial for maintaining the integrity and reliability of the entire system, regardless of the specific financial implications.

Therefore both mentioned impacts should be relevant to this bug and bug report, but at least the griefing impact if not both.

## IMPACTS:

* Even if both the community & operations multisigs were/are fully TRUSTED, this trust factor does not cover accidental actions.
* If a role/user/multisig is fully trusted, the assumption is that they will never do anything bad out of malicious intention, however, bad things could happen due to accidental actions too, and the current implementation has no mitigation against accidental transaction executions made possible by this bug.

### References

<https://github.com/PufferFinance/pufETH/blob/3e76d02c7b54323d347c8277327d3877bab591f5/src/Timelock.sol#L182-L225>

### Proof of Concept

## PoC tests:

PRIMARY PoC tests: main PoCs for this bug report TEST 1: proving the bug exists TEST 2: proving my bugfix works

SECONDARY PoC tests:

* (I will investigate the below further and if I think valid, I will do PoCs for them too, and add to this bug report or create a new report, depending on feedback from Immunefi team.): TEST 3: proving that it's possible to execute same upgrade tx twice TEST 4: proving temporary freezing of funds: might not be valid, need to check. TEST 5: proving (temporary) insolvency: might not be valid, need to check.

For this bug report and its PoC I made use of existing test file `pufETH/test/Integration/PufferTest.integration.t.sol`, and added my own test function as per below.

> Please note: CASE 1 and CASE 2 from the below test function should not be executed at the same time, so execute one of them at a time, and keep the other one commented out. See details in test function below:

## MY TEST FUNCTION:

```solidity
    /// //audit test function added for PoC/testing purposes
    function test_double_execute_queued_EL_deposit_tx()
        public
        giveToken(BLAST_DEPOSIT, address(stETH), address(pufferVault), 2000 ether) // Blast got a lot of stETH
    {
        uint256 assetsBefore = pufferVault.totalAssets();
        assertEq(assetsBefore, stETH.balanceOf(address(pufferVault)), "pufferVault stETH balance != pufferVault totalAssets");

        // 1 wei diff because of rounding
        assertApproxEqAbs(assetsBefore, 2000 ether, 1, "should have 2k ether");

        _increaseELstETHCap();

        /// @audit this block is to setup calls to `timelock.executeTransaction()`:
        uint256 assetsBefore_half = assetsBefore/2;
        bytes memory callData = abi.encodeCall(pufferVault.depositToEigenLayer, (assetsBefore_half));
        uint256 operationId = 1234;
        uint256 lockedUntil = block.timestamp + timelock.delay();

        /// CASE 1: operations multisig executes queued tx first; comment out CASE 2 before executing CASE 1.
        vm.startPrank(OPERATIONS_MULTISIG);

        timelock.queueTransaction(address(pufferVault), callData, operationId);
        vm.warp(lockedUntil + 1);

        assertEq(_EIGEN_STETH_STRATEGY.userUnderlying(address(pufferVault)), 0, "pufferVault somehow already has shares");
        
        /// EL DEPOSIT:
        timelock.executeTransaction(address(pufferVault), callData, operationId);
        uint256 ownedShares_singleDeposit = _EIGEN_STRATEGY_MANAGER.stakerStrategyShares(address(pufferVault), _EIGEN_STETH_STRATEGY);
        assertGt(ownedShares_singleDeposit, 0);
        assertLt(_EIGEN_STETH_STRATEGY.userUnderlying(address(pufferVault)), 1000 ether, "deposit to EL >= 1000");
        assertGt(_EIGEN_STETH_STRATEGY.userUnderlying(address(pufferVault)), 999 ether, "deposit to EL <= 999");
        uint256 stETH_BalanceOf_before = stETH.balanceOf(address(pufferVault));
        assertGt(stETH_BalanceOf_before, 0, "pufferVault stETH balance should be above zero still");
        vm.stopPrank();
        
        /// then let community multisig execute the queued tx too:
        vm.startPrank(COMMUNITY_MULTISIG);
        timelock.executeTransaction(address(pufferVault), callData, operationId);
        uint256 ownedShares_doubleDeposit = _EIGEN_STRATEGY_MANAGER.stakerStrategyShares(address(pufferVault), _EIGEN_STETH_STRATEGY);
        assertGt(ownedShares_doubleDeposit, ownedShares_singleDeposit);
        assertLt(_EIGEN_STETH_STRATEGY.userUnderlying(address(pufferVault)), 2000 ether, "deposit to EL >= 2000");
        assertGt(_EIGEN_STETH_STRATEGY.userUnderlying(address(pufferVault)), 1999 ether, "deposit to EL <= 1999");
        assertGt(stETH_BalanceOf_before, stETH.balanceOf(address(pufferVault)), "pufferVault stETH balance should be less now than after first deposit");
        vm.stopPrank();

        // /// CASE 2: community multisig executes queued tx first; comment out CASE 1 before executing CASE 2.
        // vm.startPrank(OPERATIONS_MULTISIG);
        // timelock.queueTransaction(address(pufferVault), callData, operationId);
        // vm.stopPrank();

        // vm.startPrank(COMMUNITY_MULTISIG);
        // /// EL DEPOSIT:
        // timelock.executeTransaction(address(pufferVault), callData, operationId);
        // uint256 ownedShares_singleDeposit = _EIGEN_STRATEGY_MANAGER.stakerStrategyShares(address(pufferVault), _EIGEN_STETH_STRATEGY);
        // assertGt(ownedShares_singleDeposit, 0);
        // assertLt(_EIGEN_STETH_STRATEGY.userUnderlying(address(pufferVault)), 1000 ether, "deposit to EL >= 1000");
        // assertGt(_EIGEN_STETH_STRATEGY.userUnderlying(address(pufferVault)), 999 ether, "deposit to EL <= 999");
        // uint256 stETH_BalanceOf_before = stETH.balanceOf(address(pufferVault));
        // assertGt(stETH_BalanceOf_before, 0, "pufferVault stETH balance should be above zero still");
        // /// Prepare for when operations multisig will call same tx accidentally/maliciously:
        // vm.warp(lockedUntil + 1);
        // vm.stopPrank();

        // /// then let operations multisig execute their queued tx too:
        // vm.startPrank(OPERATIONS_MULTISIG);
        // timelock.executeTransaction(address(pufferVault), callData, operationId);
        // uint256 ownedShares_doubleDeposit = _EIGEN_STRATEGY_MANAGER.stakerStrategyShares(address(pufferVault), _EIGEN_STETH_STRATEGY);
        // assertGt(ownedShares_doubleDeposit, ownedShares_singleDeposit);
        // assertLt(_EIGEN_STETH_STRATEGY.userUnderlying(address(pufferVault)), 2000 ether, "deposit to EL >= 2000");
        // assertGt(_EIGEN_STETH_STRATEGY.userUnderlying(address(pufferVault)), 1999 ether, "deposit to EL <= 1999");
        // assertGt(stETH_BalanceOf_before, stETH.balanceOf(address(pufferVault)), "pufferVault stETH balance should be less now than after first deposit");
        // vm.stopPrank();
    }
```

## TEST 1: proving the bug exists

forge command used: `ETH_RPC_URL=https://mainnet.infura.io/v3/84da59df4c5640e0a7da367d8fcb76b1 forge test --match-test test_double_execute_queued_EL_deposit_tx -vvvvv`

> CASE 1: operations multisig executes queued tx first:

* operations multisig queues deposit tx and then executes it after timelock delay, then community multisig successfully executes same tx again.
* Result: +-2000 stETH deposited into EL strategy contract

```solidity
    │   │   │   ├─ [2972] 0x17144556fd3424EDC8Fc8A4C940B2D04936d17eb::balanceOf(Eigen stETH strategy: [0x93c4b944D05dfe6df7645A86cd2206016c51564D]) [delegatecall]
    │   │   │   │   └─ ← 77473584173803977821625 [7.747e22]
    │   │   │   └─ ← 77473584173803977821625 [7.747e22]
    │   │   └─ ← 1999999999999999999997 [1.999e21]
    │   └─ ← 1999999999999999999997 [1.999e21]
    ├─ [6352] stETH::balanceOf(PufferVault: [0x989e351E99b9c7Ff8a31042521C772b0ECED39D3]) [staticcall]
    │   ├─ [1763] 0xb8FFC3Cd6e7Cf5a098A1c92F48009765B24088Dc::getApp(0xf1f3eb40f5bc1ad1344716ced8b8a0431d840b5783aea1fd01786bc26f35ac0f, 0x3ca7c3e38968823ccb4c78ea688df41356f182ae1d159e4ee608d30d68cef320)
    │   │   ├─ [820] 0x2b33CF282f867A7FF693A66e11B0FcC5552e4425::getApp(0xf1f3eb40f5bc1ad1344716ced8b8a0431d840b5783aea1fd01786bc26f35ac0f, 0x3ca7c3e38968823ccb4c78ea688df41356f182ae1d159e4ee608d30d68cef320) [delegatecall]
    │   │   │   └─ ← 0x00000000000000000000000017144556fd3424edc8fc8a4c940b2d04936d17eb
    │   │   └─ ← 0x00000000000000000000000017144556fd3424edc8fc8a4c940b2d04936d17eb
    │   ├─ [2972] 0x17144556fd3424EDC8Fc8A4C940B2D04936d17eb::balanceOf(PufferVault: [0x989e351E99b9c7Ff8a31042521C772b0ECED39D3]) [delegatecall]
    │   │   └─ ← 2
    │   └─ ← 2
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.44s

Ran 1 test suite in 1.44s: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

> CASE 2: community multisig executes queued tx first:

* community multisig executes queued tx before/after timelock delay expires, then operations multisig successfully executes same tx again.
* Result: +-2000 stETH deposited into EL strategy contract

```solidity
    │   │   │   ├─ [2972] 0x17144556fd3424EDC8Fc8A4C940B2D04936d17eb::balanceOf(Eigen stETH strategy: [0x93c4b944D05dfe6df7645A86cd2206016c51564D]) [delegatecall]
    │   │   │   │   └─ ← 77473584173803977821625 [7.747e22]
    │   │   │   └─ ← 77473584173803977821625 [7.747e22]
    │   │   └─ ← 1999999999999999999997 [1.999e21]
    │   └─ ← 1999999999999999999997 [1.999e21]
    ├─ [6352] stETH::balanceOf(PufferVault: [0x989e351E99b9c7Ff8a31042521C772b0ECED39D3]) [staticcall]
    │   ├─ [1763] 0xb8FFC3Cd6e7Cf5a098A1c92F48009765B24088Dc::getApp(0xf1f3eb40f5bc1ad1344716ced8b8a0431d840b5783aea1fd01786bc26f35ac0f, 0x3ca7c3e38968823ccb4c78ea688df41356f182ae1d159e4ee608d30d68cef320)
    │   │   ├─ [820] 0x2b33CF282f867A7FF693A66e11B0FcC5552e4425::getApp(0xf1f3eb40f5bc1ad1344716ced8b8a0431d840b5783aea1fd01786bc26f35ac0f, 0x3ca7c3e38968823ccb4c78ea688df41356f182ae1d159e4ee608d30d68cef320) [delegatecall]
    │   │   │   └─ ← 0x00000000000000000000000017144556fd3424edc8fc8a4c940b2d04936d17eb
    │   │   └─ ← 0x00000000000000000000000017144556fd3424edc8fc8a4c940b2d04936d17eb
    │   ├─ [2972] 0x17144556fd3424EDC8Fc8A4C940B2D04936d17eb::balanceOf(PufferVault: [0x989e351E99b9c7Ff8a31042521C772b0ECED39D3]) [delegatecall]
    │   │   └─ ← 2
    │   └─ ← 2
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.72s

Ran 1 test suite in 1.72s: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## TEST 2: proving my bugfix works

`ETH_RPC_URL=https://mainnet.infura.io/v3/84da59df4c5640e0a7da367d8fcb76b1 forge test --match-test test_double_execute_queued_EL_deposit_tx -vvvvv`

> CASE 1: operations multisig executes queued tx first:

* operations multisig queues deposit tx and then executes it after timelock delay, then community multisig tries to execute same tx again, but it reverts
* Result: only +-1000 stETH deposited into EL strategy contract

```solidity
    │   │   │   ├─ [2972] 0x17144556fd3424EDC8Fc8A4C940B2D04936d17eb::balanceOf(Eigen stETH strategy: [0x93c4b944D05dfe6df7645A86cd2206016c51564D]) [delegatecall]
    │   │   │   │   └─ ← 76473584173803977821627 [7.647e22]
    │   │   │   └─ ← 76473584173803977821627 [7.647e22]
    │   │   └─ ← 999999999999999999998 [9.999e20]
    │   └─ ← 999999999999999999998 [9.999e20]
    ├─ [6352] stETH::balanceOf(PufferVault: [0x989e351E99b9c7Ff8a31042521C772b0ECED39D3]) [staticcall]
    │   ├─ [1763] 0xb8FFC3Cd6e7Cf5a098A1c92F48009765B24088Dc::getApp(0xf1f3eb40f5bc1ad1344716ced8b8a0431d840b5783aea1fd01786bc26f35ac0f, 0x3ca7c3e38968823ccb4c78ea688df41356f182ae1d159e4ee608d30d68cef320)
    │   │   ├─ [820] 0x2b33CF282f867A7FF693A66e11B0FcC5552e4425::getApp(0xf1f3eb40f5bc1ad1344716ced8b8a0431d840b5783aea1fd01786bc26f35ac0f, 0x3ca7c3e38968823ccb4c78ea688df41356f182ae1d159e4ee608d30d68cef320) [delegatecall]
    │   │   │   └─ ← 0x00000000000000000000000017144556fd3424edc8fc8a4c940b2d04936d17eb
    │   │   └─ ← 0x00000000000000000000000017144556fd3424edc8fc8a4c940b2d04936d17eb
    │   ├─ [2972] 0x17144556fd3424EDC8Fc8A4C940B2D04936d17eb::balanceOf(PufferVault: [0x989e351E99b9c7Ff8a31042521C772b0ECED39D3]) [delegatecall]
    │   │   └─ ← 1000000000000000000000 [1e21]
    │   └─ ← 1000000000000000000000 [1e21]
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(COMMUNITY_MULTISIG: [0xfB8F2a4a911a15113D045DA118f5afA48de78aF6])
    │   └─ ← ()
    ├─ [1284] Timelock::executeTransaction(PufferVault: [0x989e351E99b9c7Ff8a31042521C772b0ECED39D3], 0x008e059000000000000000000000000000000000000000000000003635c9adc5de9fffff, 1234)
    │   └─ ← InvalidTransaction(0x53f339d06d7c7d37d5707e46492448946eec4cc056d22a35403dbad67c29294b)
    └─ ← InvalidTransaction(0x53f339d06d7c7d37d5707e46492448946eec4cc056d22a35403dbad67c29294b)

Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.71s

Ran 1 test suite in 1.71s: 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/Integration/PufferTest.integration.t.sol:PufferTest
[FAIL. Reason: InvalidTransaction(0x53f339d06d7c7d37d5707e46492448946eec4cc056d22a35403dbad67c29294b)] test_double_execute_queued_EL_deposit_tx() (gas: 623412)

Encountered a total of 1 failing tests, 0 tests succeeded
```

> CASE 2: community multisig executes queued tx first:

* community multisig executes queued tx before/after timelock delay expires, then operations multisig tries to execute same tx again, but it reverts.
* Result: only +-1000 stETH deposited into EL strategy contract

```solidity
    │   │   │   ├─ [2972] 0x17144556fd3424EDC8Fc8A4C940B2D04936d17eb::balanceOf(Eigen stETH strategy: [0x93c4b944D05dfe6df7645A86cd2206016c51564D]) [delegatecall]
    │   │   │   │   └─ ← 76473584173803977821627 [7.647e22]
    │   │   │   └─ ← 76473584173803977821627 [7.647e22]
    │   │   └─ ← 999999999999999999998 [9.999e20]
    │   └─ ← 999999999999999999998 [9.999e20]
    ├─ [6352] stETH::balanceOf(PufferVault: [0x989e351E99b9c7Ff8a31042521C772b0ECED39D3]) [staticcall]
    │   ├─ [1763] 0xb8FFC3Cd6e7Cf5a098A1c92F48009765B24088Dc::getApp(0xf1f3eb40f5bc1ad1344716ced8b8a0431d840b5783aea1fd01786bc26f35ac0f, 0x3ca7c3e38968823ccb4c78ea688df41356f182ae1d159e4ee608d30d68cef320)
    │   │   ├─ [820] 0x2b33CF282f867A7FF693A66e11B0FcC5552e4425::getApp(0xf1f3eb40f5bc1ad1344716ced8b8a0431d840b5783aea1fd01786bc26f35ac0f, 0x3ca7c3e38968823ccb4c78ea688df41356f182ae1d159e4ee608d30d68cef320) [delegatecall]
    │   │   │   └─ ← 0x00000000000000000000000017144556fd3424edc8fc8a4c940b2d04936d17eb
    │   │   └─ ← 0x00000000000000000000000017144556fd3424edc8fc8a4c940b2d04936d17eb
    │   ├─ [2972] 0x17144556fd3424EDC8Fc8A4C940B2D04936d17eb::balanceOf(PufferVault: [0x989e351E99b9c7Ff8a31042521C772b0ECED39D3]) [delegatecall]
    │   │   └─ ← 1000000000000000000000 [1e21]
    │   └─ ← 1000000000000000000000 [1e21]
    ├─ [0] VM::warp(1703593296 [1.703e9])
    │   └─ ← ()
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(OPERATIONS_MULTISIG: [0x78dE5808728A273648A7D301D7767C4fd5Dc0fF6])
    │   └─ ← ()
    ├─ [1336] Timelock::executeTransaction(PufferVault: [0x989e351E99b9c7Ff8a31042521C772b0ECED39D3], 0x008e059000000000000000000000000000000000000000000000003635c9adc5de9fffff, 1234)
    │   └─ ← InvalidTransaction(0x53f339d06d7c7d37d5707e46492448946eec4cc056d22a35403dbad67c29294b)
    └─ ← InvalidTransaction(0x53f339d06d7c7d37d5707e46492448946eec4cc056d22a35403dbad67c29294b)

Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.90s

Ran 1 test suite in 1.90s: 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/Integration/PufferTest.integration.t.sol:PufferTest
[FAIL. Reason: InvalidTransaction(0x53f339d06d7c7d37d5707e46492448946eec4cc056d22a35403dbad67c29294b)] test_double_execute_queued_EL_deposit_tx() (gas: 609793)

Encountered a total of 1 failing tests, 0 tests succeeded
```

## SUGGESTED BUG MITIGATION / BUGFIX:

My suggested bugfixes below to prevent queued transaction from being successfully executed twice:

```diff
    function executeTransaction(address target, bytes calldata callData, uint256 operationId) 
        external
        returns (bool success, bytes memory returnData)
    {
+    	/// @audit Moved here so that the community multisig `if` block below can use it
+    	bytes32 txHash = keccak256(abi.encode(target, callData, operationId));
+    	uint256 lockedUntil = queue[txHash];
    	
        // Community Multisig can do things without any delay
        if (msg.sender == COMMUNITY_MULTISIG) {
+        	// slither-disable-next-line incorrect-equality
+        	if (lockedUntil == 0) {
+            	revert InvalidTransaction(txHash); /// @audit tx already executed (or operationId/txHash is invalid).
+        	}
+			queue[txHash] = 0; /// @audit added here to ensure we cover both sides
            return _executeTransaction(target, callData);
        }

        // Operations multisig needs to queue it and then execute after a delay
        if (msg.sender != OPERATIONS_MULTISIG) {
            revert Unauthorized();
        }

-       bytes32 txHash = keccak256(abi.encode(target, callData, operationId));
-       uint256 lockedUntil = queue[txHash];

        // slither-disable-next-line incorrect-equality
        if (lockedUntil == 0) {
            revert InvalidTransaction(txHash);
        }

        if (block.timestamp < lockedUntil) {
            revert Locked(txHash, lockedUntil);
        }

        queue[txHash] = 0;
        (success, returnData) = _executeTransaction(target, callData);

        emit TransactionExecuted(txHash, target, callData, operationId);

        return (success, returnData);
    }
```
