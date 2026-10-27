# Boost \_ Folks Finance 33356 - \[Smart Contract - Low] All data in \_userLoans mapping will not be dele

Submitted on Thu Jul 18 2024 15:57:02 GMT-0400 (Atlantic Standard Time) by @Lastc0de for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33356

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In Solidity, a struct is a complex data type that allows you to group together variables of different data types. And a mapping is a data structure that allows you to store key-value pairs.

The security implications of deleting a struct that contains a mapping are subtle, but important to understand in the context of Ethereum smart contracts.

When you delete a struct in Solidity, it will not delete the mapping within it. The delete keyword in Solidity sets every field in the struct to its default value. For integers, strings, arrays, and other simple data types, this means they will be set to zero, an empty string, or an empty array, respectively.

However, for mappings, the delete keyword has no effect. This is because mappings are implemented as hash tables and the Ethereum Virtual Machine (EVM) does not keep track of which keys have been used in the mapping. As a result, it doesn't know how to "reset" a mapping. Therefore, when you delete a struct, the mapping within it will still retain its old data.

This can lead to potential security issues, particularly if you’re not aware of this behavior. For example, let’s say you have a struct that contains sensitive data within a mapping. If you delete the struct assuming that all data within it will be erased, the data in the mapping will still persist, potentially leading to unintended access or misuse.

## Vulnerability Details

unintended access or misuse.

* Vulnerable contract is `LoanManager.sol` : <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/LoanManager.sol>
* Vulnerable function is `deleteUserLoan()` : <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/LoanManager.sol#L60C1-L73C1>

```
    function deleteUserLoan(bytes32 loanId, bytes32 accountId) external override onlyRole(HUB_ROLE) nonReentrant {
        // check user loan active and account owner
        if (!isUserLoanActive(loanId)) revert UnknownUserLoan(loanId);
        if (!isUserLoanOwner(loanId, accountId)) revert NotAccountOwner(loanId, accountId);

        // ensure loan is empty
        if (!_isUserLoanEmpty(loanId)) revert LoanNotEmpty(loanId);

        // delete by setting isActive to false
        delete _userLoans[loanId]; // @Audit is here

        emit DeleteUserLoan(loanId, accountId);
    }
```

This function will delete `_userLoans` after several checks. `_userLoans` is struct which contains a mapping.

```
    mapping(bytes32 loanId => UserLoan) internal _userLoans;
```

`UserLoan` struct have two struct which contains mapping - `UserLoanCollateral` and `UserLoanBorrow`:

```
    struct UserLoan {
        bool isActive;
        bytes32 accountId;
        uint16 loanTypeId;
        uint8[] colPools;
        uint8[] borPools;
        mapping(uint8 poolId => UserLoanCollateral) collaterals; // @audit is here
        mapping(uint8 poolId => UserLoanBorrow) borrows; // Audit is here
    }
```

`UserLoanCollateral` and `UserLoanBorrow` , each of these two stores important values:

```
    struct UserLoanCollateral {
        uint256 balance; // denominated in f token
        uint256 rewardIndex;
    }

    struct UserLoanBorrow {
        uint256 amount; // excluding interest
        uint256 balance; // including interest
        uint256 lastInterestIndex;
        uint256 stableInterestRate; // defined if stable borrow
        uint256 lastStableUpdateTimestamp; // defined if stable borrow
        uint256 rewardIndex;
    }
```

**Result:** The `deleteUserLoan` function used **delete** keyword to delete `_userLoans` struct which contains mapping for a `loanId`. However, if you call the `deleteUserLoan` function, it will not delete the **collaterals** and **borrows** mapping within the UserInfo struct. This means that even after a user has been deleted, their active data will still persist in the contract.

## Impact Details

All data in \_userLoans for loanId is not deleted we expect it to be completely erased

## References

Add any relevant links to documentation or code

## Proof of concept

## Proof of Concept

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";


contract FolksStructFuzzing is Test {
    // Structs forked from : https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/LoanManagerState.sol#L65C1-L93C2
    struct UserLoanCollateral {
        uint256 balance; // denominated in f token
        uint256 rewardIndex;
    }

    struct UserLoanBorrow {
        uint256 amount; // excluding interest
        uint256 balance; // including interest
        uint256 lastInterestIndex;
        uint256 stableInterestRate; // defined if stable borrow
        uint256 lastStableUpdateTimestamp; // defined if stable borrow
        uint256 rewardIndex;
    }


    struct UserLoan {
        bool isActive;
        bytes32 accountId;
        uint16 loanTypeId;
        uint8[] colPools;
        uint8[] borPools;
        mapping(uint8 poolId => UserLoanCollateral) collaterals;
        mapping(uint8 poolId => UserLoanBorrow) borrows;
    }

    // #################################################################################

    // Mapping forked from : https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/LoanManagerState.sol#L100
    mapping(bytes32 loanId => UserLoan) public _userLoans;


    bytes32 testloanId32 = 0x6b62515a7648fc480b4b4c3543f68573506917c5060ae74240cf97d70165cbe7;
    bytes32 testaccountId32 = 0x5552515a7648fc480b4b4c3543f68573506917c5060ae74240cf97d70165def6;
    bytes32 testLoanName = 0x5552515a7648fc480b4b4c2223f68573506917c7777ae74240cf97d70165def6;

    function setUp() public {}

    function test_Fuzzing_Deleting_Struct_Mapping() public {
        // STEP-1 create user loan --> `testloanId32`
        createUserLoan(testloanId32,testaccountId32,1,testLoanName);

        // STEP-2 Log _userLoans[testloanId32] mapping before deleted
        logStructStorage("LOG - _userLoans[testloanId32] BEFORE Deleting",testloanId32);

        // STEP-3 delete User Loan for --> `testloanId32`
        deleteUserLoan(testloanId32,testaccountId32);

        // STEP-2 Log _userLoans[testloanId32] mapping after deleted
        console.log("\n");
        logStructStorage("LOG - _userLoans[testloanId32] AFTER Deleting",testloanId32);

    }
    // createUserLoan() function forked from : https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/LoanManager.sol#L40C1-L58C6
    function createUserLoan(
        bytes32 loanId,
        bytes32 accountId,
        uint16 loanTypeId,
        bytes32 loanName
        ) internal {
        // create loan
        UserLoan storage userLoan = _userLoans[loanId];
        userLoan.isActive = true;
        userLoan.accountId = accountId;
        userLoan.loanTypeId = loanTypeId;
        userLoan.colPools = [1];
        userLoan.borPools = [10];
        userLoan.collaterals[0].balance = 100e6;
        userLoan.collaterals[0].rewardIndex = 26e6;
        userLoan.borrows[0].amount = 100e6;
        userLoan.borrows[0].balance = 100e6;
        userLoan.borrows[0].lastInterestIndex = 12345456775444;
        userLoan.borrows[0].stableInterestRate = 1e32;
        userLoan.borrows[0].lastStableUpdateTimestamp = block.timestamp;
        userLoan.borrows[0].rewardIndex = 26e6;

    }

    // deleteUserLoan() function forked from : https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/LoanManager.sol#L60C1-L73C1
    function deleteUserLoan(bytes32 loanId, bytes32 accountId) internal {
        delete _userLoans[loanId]; // @audit
    }



    function logStructStorage(string memory str,bytes32 loanId) internal {
        UserLoan storage userLoan = _userLoans[loanId];
        console.log(str);
        console.log(" userLoan.isActive = %s", userLoan.isActive);
        console.log(" userLoan.loanTypeId = ", userLoan.loanTypeId);
        console.log();
        console.log("Deleting structs which contains a mapping\n");

        console.log(" userLoan.collaterals[0].balance = ", userLoan.collaterals[0].balance);
        console.log(" userLoan.collaterals[0].rewardIndex = ", userLoan.collaterals[0].rewardIndex);
        console.log(" userLoan.borrows[0].amount = ", userLoan.borrows[0].amount);
        console.log(" userLoan.borrows[0].balance = ", userLoan.borrows[0].balance);
        console.log(" userLoan.borrows[0].lastInterestIndex = ", userLoan.borrows[0].lastInterestIndex);
        console.log(" userLoan.borrows[0].stableInterestRate = ", userLoan.borrows[0].stableInterestRate);
        console.log(" userLoan.borrows[0].lastStableUpdateTimestamp = ", userLoan.borrows[0].lastStableUpdateTimestamp);
        console.log(" userLoan.borrows[0].rewardIndex = ", userLoan.borrows[0].rewardIndex);
    }
}

```
