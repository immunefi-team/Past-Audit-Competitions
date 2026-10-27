# 28991 - \[SC - Insight] Contract uint delay variable cannot be set to i...

Submitted on Mar 4th 2024 at 04:53:59 UTC by @Obin for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28991

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The Timelock contract intends to alter its `uint delay` variable via the `setDelay()` which take an input of a predefined range of numbers. However, the implementation in this `setDelay()` is not accurate and doesn't allow to set the minimum acceptable value.

```
function _setDelay(uint256 newDelay) internal {
        if (newDelay <= MINIMUM_DELAY) {
            revert InvalidDelay(newDelay);
        }
        emit DelayChanged(delay, newDelay);
        delay = newDelay;
    }
```

## Impact Details

`uint delay` variable cannot be set to its minimum acceptable value `uint256 public constant MINIMUM_DELAY = 7 days;`

## Mitigation

```diff
function _setDelay(uint256 newDelay) internal {
-       if (newDelay <= MINIMUM_DELAY) {
+       if (newDelay < MINIMUM_DELAY) {
            revert InvalidDelay(newDelay);
        }
        emit DelayChanged(delay, newDelay);
        delay = newDelay;
    }
```

## Proof of Concept

```//
pragma solidity >=0.8.0 <0.9.0;

import { Test } from "forge-std/Test.sol";
import { console2 } from "forge-std/console2.sol";

interface Timelock {
    event DelayChanged(uint256 oldDelay, uint256 newDelay);
    event PauserChanged(address oldPauser, address newPauser);
    event TransactionCanceled(
        bytes32 indexed txHash, address indexed target, bytes callData, uint256 indexed operationId
    );
    event TransactionExecuted(
        bytes32 indexed txHash, address indexed target, bytes callData, uint256 indexed operationId
    );
    event TransactionQueued(
        bytes32 indexed txHash, address indexed target, bytes callData, uint256 indexed operationId, uint256 lockedUntil
    );

    function ACCESS_MANAGER() external view returns (address);
    function COMMUNITY_MULTISIG() external view returns (address);
    function MINIMUM_DELAY() external view returns (uint256);
    function OPERATIONS_MULTISIG() external view returns (address);
    function cancelTransaction(address target, bytes memory callData, uint256 operationId) external;
    function delay() external view returns (uint256);
    function executeTransaction(address target, bytes memory callData, uint256 operationId)
        external
        returns (bool success, bytes memory returnData);
    function pause(address[] memory targets) external;
    function pauserMultisig() external view returns (address);
    function queue(bytes32 transactionHash) external view returns (uint256 lockedUntil);
    function queueTransaction(address target, bytes memory callData, uint256 operationId) external returns (bytes32);
    function setDelay(uint256 newDelay) external;
    function setPauser(address newPauser) external;
}


contract TimelockTest is Test { 
    string internal mainnetRpc = "https://eth-mainnet.g.alchemy.com/v2/CZIEYE1tyZTnLXtG8e6wh2YboDUPJ43f";
    address public timeLockAddr = 0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA;
    uint blockId;
    Timelock timelock;

    function setUp() public {
        blockId = vm.createSelectFork(mainnetRpc);
        timelock = Timelock(timeLockAddr);
    }

    function testMINIMUM_DELAY_Cannnot_be_set() public {
        
        
        //address Msig
        address OPERATIONS_MULTISIG = timelock.OPERATIONS_MULTISIG();
        bytes memory expectedReturnData = hex"4c89d5980000000000000000000000000000000000000000000000000000000000093a80"; //bytes(revert In validDelay(newDelay))

        //make Tx for queue (intends to setDelay at min value)
        uint operationId = 444;
        bytes memory KallData = abi.encodeCall(Timelock.setDelay, (timelock.MINIMUM_DELAY()));
        
        //Queue Tx
        vm.startPrank(OPERATIONS_MULTISIG);
        bytes32 TxHash = timelock.queueTransaction(address(timelock),KallData,operationId);
        
        
        uint LockedUntil = timelock.queue(TxHash);
        console2.log("LockedUntil",LockedUntil);
        assertTrue(LockedUntil != 0);

        vm.warp(block.timestamp + timelock.delay());
        
        (bool succes, bytes memory retValue ) = timelock.executeTransaction(address(timelock),KallData,operationId);
        console2.log("succes",succes);

        assertTrue(succes == false); //
        assertEq(retValue , expectedReturnData, "not same");
        
        
    }

}
```
