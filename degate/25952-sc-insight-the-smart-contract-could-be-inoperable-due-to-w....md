# 25952 - \[SC - Insight] The smart contract could be inoperable due to w\...

Submitted on Nov 21st 2023 at 22:07:40 UTC by @piken for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25952

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52#code>

Impacts:

* The smart contract could be inoperable due to wrong replaceOwner() calling

## Description

## Bug Description

`MultiSigWallet` allows eligible owner to submit any transaction by calling `submitTransaction()`:

```solidity
    function submitTransaction(address destination, uint value, bytes data)
        public
        returns (uint transactionId)
    {
        transactionId = addTransaction(destination, value, data);
        confirmTransaction(transactionId);
    }
```

Other owners can confirm the transaction by calling `confirmTransaction()`:

```solidity
    function confirmTransaction(uint transactionId)
        public
        ownerExists(msg.sender)
        transactionExists(transactionId)
        notConfirmed(transactionId, msg.sender)
    {
        confirmations[transactionId][msg.sender] = true;
        Confirmation(msg.sender, transactionId);
        executeTransaction(transactionId);
    }
```

The transaction will be executed once confirmation numbers reach the threshold `required`

```solidity
    function executeTransaction(uint transactionId)
        public
        notExecuted(transactionId)
    {
        if (isConfirmed(transactionId)) {
            Transaction tx = transactions[transactionId];
            tx.executed = true;
            if (tx.destination.call.value(tx.value)(tx.data))
                Execution(transactionId);
            else {
                ExecutionFailure(transactionId);
                tx.executed = false;
            }
        }
    }
    function isConfirmed(uint transactionId)
        public
        constant
        returns (bool)
    {
        uint count = 0;
        for (uint i=0; i<owners.length; i++) {
            if (confirmations[transactionId][owners[i]])
                count += 1;
            if (count == required)
                return true;
        }
    }
```

The owner of `MultiSigWallet` can also be replaced through above process. However, the owner could be replaced to `address(0)` if somehow the owners made a mistake. Additionally, if the total number of owners equals to threshold `required` before incorrect owner replacing, there would be no enough qualified owners to correct the mistake after the transaction was executed.

## Impact

1. `MultiSigWallet` will be inoperable
2. Any assets in `MultiSigWallet` will be locked forever

## Risk Breakdown

Difficulty to Exploit: Hard

## Recommendation

Check if `newOwner` is `address(0)`:

```diff
    function replaceOwner(address owner, address newOwner)
        public
        onlyWallet
        ownerExists(owner)
        ownerDoesNotExist(newOwner)
+       notNull(newOwner)
    {
        for (uint i=0; i<owners.length; i++)
            if (owners[i] == owner) {
                owners[i] = newOwner;
                break;
            }
        isOwner[owner] = false;
        isOwner[newOwner] = true;
        OwnerRemoval(owner);
        OwnerAddition(newOwner);
    }
```

## References

<https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52#code>

Line165\~Line180

## Proof of concept

1. Create a foundry project
2. Copy below codes to test/MultiSigWallet.t.sol
3. Run `forge test --fork-url MAINNET_URL --chain-id 1 --fork-block-number 18620472 --match-test testUpdateOwner`(replace MAINNET\_URL with your Ethereum RPC URL to test:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
interface IMultiSigWallet {
    function submitTransaction(address destination, uint value, bytes memory data) external returns (uint transactionId);
    function confirmTransaction(uint transactionId) external;
    function revokeConfirmation(uint transactionId) external;
    function executeTransaction(uint transactionId) external;
    function getOwners() external returns (address[] memory);
    function required() external returns (uint);
    function isOwner(address owner) external returns (bool);
}

contract MultiSigWalletTest is Test {
    IMultiSigWallet wallet;
    function setUp() public {
        wallet = IMultiSigWallet(0x2028834B2c0A36A918c10937EeA71BE4f932da52);
    }
    function testUpdateOwner() public {
        address[] memory owners = wallet.getOwners();
        assertEq(owners.length, 6);
        assertEq(wallet.required(),4);

        address destination = address(wallet);
        uint value = 0;
        bytes memory removeOwnerData1 = abi.encodeWithSignature("removeOwner(address)", owners[4]);
        bytes memory removeOwnerData2 = abi.encodeWithSignature("removeOwner(address)", owners[5]);

        //remove 2 owners 
        vm.startPrank(owners[0]);
        uint transactionId1 = wallet.submitTransaction(destination, value, removeOwnerData1);
        uint transactionId2 = wallet.submitTransaction(destination, value, removeOwnerData2);
        vm.stopPrank();
        for (uint i=1; i<4; i++) {
            vm.startPrank(owners[i]);
            wallet.confirmTransaction(transactionId1);
            wallet.confirmTransaction(transactionId2);
            vm.stopPrank();
        }
        owners = wallet.getOwners();
        //the total number of owners is 4, equals to the threshold `required`
        assertEq(owners.length, 4);
        assertEq(wallet.required(),4);
        //replace the last owner to address(0)
        bytes memory replaceOwnerData = abi.encodeWithSignature("replaceOwner(address,address)",owners[3], address(0));
        vm.prank(owners[0]);
        uint transactionId = wallet.submitTransaction(destination, value, replaceOwnerData);
        for (uint i=1; i<4; i++) {
            vm.prank(owners[i]);
            wallet.confirmTransaction(transactionId);
        }
        //the last owner became address(0), threshold i still 4 and only 3 available owners left.afterwards the threshold can never be met
        owners = wallet.getOwners();
        assertEq(owners.length, 4);
        assertEq(wallet.required(),4);
        assertEq(owners[3], address(0));
        assertEq(wallet.isOwner(address(0)), true);
    }
}
```
