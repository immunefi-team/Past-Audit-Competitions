# 26116 - \[SC - Insight] The MultiSigWalletgetTransactionIds function co...

Submitted on Nov 25th 2023 at 12:28:13 UTC by @OxSCSamurai for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26116

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52#code>

Impacts:

* Function does not return expected value(s) due to logic bug
* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

The MultiSigWallet::getTransactionIds() function contains logic bugs which would exclude one valid transaction ID from the defined range in the returned list.

The getTransactionIds() function is designed to return an array of transaction IDs within a specified range. However, there are logic errors in the function which would result in the function returning a value(s) which is not expected, but the caller of the function will not know any better, and just assume the returned value is correct, or complete, which it isn't.

The first logic error occurs when calculating/assigning the length of the return array `_transactionIds`. Instead of using `((to + 1) - from)` to represent the correct number of entries(and array length), the statement on L373 incorrectly uses `(to - from)`: `_transactionIds = new uint[](to - from);`

This will result in an incorrect array length every time, and therefore exclude one of the valid entries that was supposed to be returned.

To demonstrate the logic bug/error: Lets say `from = 0` and `to = 5`, then the correct number of entries should be 6. But why you ask? Here you go: 0,1,2,3,4,5 = SIX entries. Therefore the array length should be 6. However, (to - from) = (5 - 0) = 5, which is 1 less than 6.

The second logic error is here on L374: `for (i=from; i<to; i++)` It excludes a valid index/entry, which is represented by the value of `to`. Yes, usually we do `i < array.length`, so what is the correct length in this example? Remember, it is 6. So if we do `i<6` then it would be correct, but the current implementation uses `i<to` which is `i<5`, which is incorrect, and will exclude the final entry, the 6th entry from the returned results.

One last time: 0,1,2,3,4,5 represents 6 entries, not 5, even though 5-0=5 OR 1,2,3,4,5,6 represents 6 entries, not 5, even though 6-1=5 OR 2,3,4,5,6,7 represents 6 entries, not 5, even though 7-2=5

Here's the function as its currently implemented:

```solidity
    function getTransactionIds(uint from, uint to, bool pending, bool executed)
        public
        constant
        returns (uint[] _transactionIds)
    {
        uint[] memory transactionIdsTemp = new uint[](transactionCount);
        uint count = 0;
        uint i;
        for (i=0; i<transactionCount; i++)
            if (   pending && !transactions[i].executed
                || executed && transactions[i].executed)
            {
                transactionIdsTemp[count] = i;
                count += 1;
            }
        _transactionIds = new uint[](to - from);
        for (i=from; i<to; i++)
            _transactionIds[i - from] = transactionIdsTemp[i];
    }
```

## Impact

* local impact on the function's returned result is that it will ALWAYS exclude one valid entry, the last entry, represented by to in the defined range.
* currently not sure of impact on protocol/users, as this is a constant/view only function it seems, so it only returns a value, I will need to check with the devs if there is any significant protocol dependencies that depend on the return value of this function, and if there are indeed critical dependencies, we might have to upgrade the severity level to high, depending.

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: CVSS2 Score:

## Recommendation

```diff
    /// @dev Returns list of transaction IDs in defined range. /// @audit the defined range is from `from` to `to`
    /// @param from Index start position of transaction array. /// @audit for case example lets say `from` is 0
    /// @param to Index end position of transaction array. /// @audit for case example lets say `to` is 5
    /// @param pending Include pending transactions.
    /// @param executed Include executed transactions.
    /// @return Returns array of transaction IDs.
    function getTransactionIds(uint from, uint to, bool pending, bool executed)
        public
        constant
        returns (uint[] _transactionIds)
    {
        uint[] memory transactionIdsTemp = new uint[](transactionCount);
        uint count = 0;
        uint i;
        for (i=0; i<transactionCount; i++)
            if (   pending && !transactions[i].executed
                || executed && transactions[i].executed)
            {
                transactionIdsTemp[count] = i;
                count += 1;
            }
-       _transactionIds = new uint[](to - from);
+       _transactionIds = new uint[]((to + 1) - from);
-       for (i=from; i<to; i++)
+       for (i=from; i<=to; i++) /// @audit OR alternatively: `for (i=from; i<to+1; i++)`
            _transactionIds[i - from] = transactionIdsTemp[i];
    }
```

## References

## Proof of concept

(Please note: My experience with foundry tests is new, so please make sure you understand my tests correctly, especially how I constructed them. I added comments in the test contract in the test functions to help you to understand my logic and method/approach. Please make sure to read those comments to avoid any confusion. Feel free to give me a shout if you need clarifications/explanations.)

I added the fixed function to the multisig wallet contract temporarily so that I could run my PoC tests to test my fix too, not just the logic bug.

Here it is, and then below the fixed function is my test contract, I used foundry, with some magic...

```solidity
    /// audit added this function temporarily for PoC testing purposes only.
    function fixed_getTransactionIds(uint from, uint to, bool pending, bool executed)
        public
        view
        returns (uint[] memory _transactionIds)
    {
        uint[] memory transactionIdsTemp = new uint[](transactionCount);
        uint count = 0;
        uint i;
        for (i=0; i<transactionCount; i++)
            if (   pending && !transactions[i].executed
                || executed && transactions[i].executed)
            {
                transactionIdsTemp[count] = i;
                count += 1;
            }
       _transactionIds = new uint[]((to + 1) - from);
       for (i=from; i<=to; i++) /// OR alternatively: `for (i=from; i<to+1; i++)`
            _transactionIds[i - from] = transactionIdsTemp[i];
    }
```

***

TEST CONTRACT:

***

```solidity
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import {MultiSigWallet} from "src/MultiSigWallet.sol";

contract TestMultiSigWallet is Test {

    MultiSigWallet multiSigWallet;

    function setUp() public {
        /// prepping the parameter values for the constructor of the MultiSigWallet. making the deployer address an owner, because why not, for this test should be valid.
        address[] memory _owners = new address[](1);
        uint _required;
        (_owners[0], _required) = (0xb4c79daB8f259C7Aee6E5b2Aa729821864227e84, 1);
        multiSigWallet = new MultiSigWallet(_owners, _required);
        /// call addTransaction() 6 times to prepare the correct contract state required for the test, since the test uses 6 transactions to prove the logic bug. Any number of transactions can be used(just adapt the test setup accordingly), but I decided on 6 transactions.
        uint[] memory _value = new uint[](6);
        _value[0] = 5;
        _value[1] = 10;
        _value[2] = 15;
        _value[3] = 20;
        _value[4] = 25;
        _value[5] = 30;
        bytes[] memory _data = new bytes[](6);
        _data[0] = hex"14521453095c890da6f48fdf14b5479a59f067f9c7927f68e66534b095c19d7c";
        _data[1] = hex"1154c9da6f65456dc8b32ce8ff50c866f00d999a4c408351cbb71458c12106b0";
        _data[2] = hex"1a6d270af40a4838666d640496d0650afb3953c610f2c8bda784c3f2aca85409";
        _data[3] = hex"26459a1be94a260b130b7b7cc18f95b6b42de3441e421a92655ed3d1020802ec";
        _data[4] = hex"11b35cf2c85531eab64b96eb2eef487e0eb60fb9207fe4763e7f6e02dcead646";
        _data[5] = hex"2cbea52f3417b398aed9e355ed16934a81b72d2646e3bf90dbc2dcba294b631d";
        /// to ensure we're working with arrays of equal length
        assert(_value.length == _data.length);
        multiSigWallet.submitTransaction(address(1), _value[0], _data[0]);
        multiSigWallet.submitTransaction(address(2), _value[1], _data[1]);
        multiSigWallet.submitTransaction(address(3), _value[2], _data[2]);
        multiSigWallet.submitTransaction(address(4), _value[3], _data[3]);
        multiSigWallet.submitTransaction(address(5), _value[4], _data[4]);
        multiSigWallet.submitTransaction(address(6), _value[5], _data[5]);
        /// to ensure that our expectation of 6 added transactions using arrays of length 6, is met. we expect `transactionCount == 6`
        uint _transactionCount = multiSigWallet.transactionCount();
        assert(_transactionCount == _value.length && _data.length == 6);
        assert(_transactionCount == 6); /// overboard additional step to quadruple check our expectations
        /// now we're all set & ready for the actual logic bug test below.
    }

    /// To demonstrate the logic bug consequences: we dont get what we expected from return value
    function test_getTransactionIds() external {
        /// setup:
        /// we're only testing executed transactions here, because it doesnt matter for the test if its pending or executed or both, as long as we include ALL the transactions in the range, specifically for this test, to eliminate any uncertainty and unexpected results. And because I dont know how to make transactions pending! :P
        /// using: `from = 0`, `to = 5`, `pending = true`, `executed = true`. Using `true` instead of `false` to ensure we dont exclude any pending/executed transactions in the given index range from 0 to 5.
        (uint _from, uint _to, bool _pending, bool _executed) = (0, 5, true, true);
        /// action: call getTransactionIds()
        uint[] memory transactionIds_ = multiSigWallet.getTransactionIds(_from, _to, _pending, _executed);
        /// asserts:

        assertEq(6, transactionIds_.length); /// if this passes, my finding is invalid. if this fails, my finding is valid.
        console.log('Left side(what it should be):', 6, "Right side(transactionIds_.length):", transactionIds_.length);

        // assertEq(5, transactionIds_.length); /// if this passes, my finding is valid. if this fails, my finding is invalid.
        // console.log('Left side(what we expect):', 5, "Right side(transactionIds_.length):", transactionIds_.length);
    }

    /// Here the logic bug is fixed, so we will get what we expected from return result
    function test_FIXED_getTransactionIds() external {
        /// setup:
        /// we're only testing executed transactions here, because it doesnt matter for the test if its pending or executed or both, as long as we include ALL the transactions in the range, specifically for this test, to eliminate any uncertainty and unexpected results. And because I dont know how to make transactions pending! :P
        /// using: `from = 0`, `to = 5`, `pending = true`, `executed = true`. Using `true` instead of `false` to ensure we dont exclude any pending/executed transactions in the given index range from 0 to 5.
        (uint _from, uint _to, bool _pending, bool _executed) = (0, 5, true, true);
        /// action: call getTransactionIds()
        uint[] memory transactionIds_ = multiSigWallet.fixed_getTransactionIds(_from, _to, _pending, _executed);
        /// asserts:

        assertEq(5, transactionIds_.length); /// if this passes, my fix is not working. if this fails, my fix is working.
        console.log('Left side:', 5, "Right side(transactionIds_.length):", transactionIds_.length);

        // assertEq(6, transactionIds_.length); /// if this passes, my fix is working. if this fails, my fix is not working.
        // console.log('Left side:', 6, "Right side(transactionIds_.length):", transactionIds_.length);
    }
}
```
