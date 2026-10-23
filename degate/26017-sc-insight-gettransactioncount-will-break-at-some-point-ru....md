# 26017 - \[SC - Insight] getTransactionCount will break at some point ru...

Submitted on Nov 23rd 2023 at 02:11:00 UTC by @dontonka for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26017

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52#code>

Impacts:

* Unbounded gas consumption

## Description

## Bug Description

`getTransactionCount` vulnerable to `run out of gas and DoS` even if it's a view function. So whatever clients rely on this view function would not be able todo his job anymore as calling this function would always revert.

## Impact

`getTransactionCount` will always revert once `~1300 transactions` (in Remix the gas limit seems to be 3M) have been submitted throught the wallet. The gas limit seems to be 30M thought, I'm not sure why in Remix it revert at 3M, but if it needs to reach 30M, then the cap is around 12000 transactions (see PoC)

## Recommendation

The problem is the for loop `for (uint i=0; i<transactionCount; i++)` which will iterate over all the txs submitted. Even if it's only to update a counter, it will exhaust all the gas available at some point, and this seems to be around having submitted 1300 transactions.

Similar to what `getTransactionIds` is doing, you would need to introduce a `range of ids` you want (`uint from, uint to`), and ensure the for loop is `only` considering this range.

```
    /// @dev Returns total number of transactions after filers are applied.
    /// @param pending Include pending transactions.
    /// @param executed Include executed transactions.
    /// @return Total number of transactions after filters are applied.
    function getTransactionCount(bool pending, bool executed)
        public
        constant
        returns (uint count)
    {
        for (uint i=0; i<transactionCount; i++)
            if (   pending && !transactions[i].executed
                || executed && transactions[i].executed)
                count += 1;
    }
```

## Proof of concept

You will have to clone the immunefi poc repo and follow the steps below. I had to upgrade the MultiSigWallet code to match solc v0.8.19 compiler in order todo this PoC. Nevertheles, I also tested in Remix using the same compiler (v0.4.26, also using the original code) and the it start reverting at around 1300 txs. Once you run the test, you can see the `gas used` which is above 30M.

* git clone <https://github.com/immunefi-team/forge-poc-templates.git>
* forge init --template immunefi-team/forge-poc-templates --branch default
* Add the following code in src folder, which is the `MultiSigWallet.sol` contract but upgraded to compile with `Solc 0.8.19`.
* Replace test/PoCTest.sol with the following test.
* forge test -vv --match-test testAttack

```
Running 1 test for test/PoCTest.sol:PoCTest
[PASS] testAttack() (gas: 30451777)
Logs:

>>> Initial conditions: # transaction submitted --> 12000

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 407.08ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "@immunefi/src/PoC.sol";
import "../src/MultiSigWallet.sol";

contract PoCTest is PoC {
    MultiSigWallet public walletContract;
    address constant ownerAccount1 = 0x2385D7aB31F5a470B1723675846cb074988531da;
    address constant ownerAccount2 = 0x0000000000000000000000000000000000000001;

    function setUp() public {
        deal(ownerAccount1, 1 ether);
        deal(ownerAccount2, 1 ether);

        // Deploy attack contract
        address[] memory accounts = new address[](2);
        accounts[0] = ownerAccount1;
        accounts[1] = ownerAccount2;
        walletContract = new MultiSigWallet(accounts, 2);
        
        // add dummy txs
        bytes memory localBytes = new bytes(1);
        for (uint k=0; k<12000; k++) {
            vm.prank(ownerAccount1);
            walletContract.submitTransaction(accounts[0], 0, localBytes);
        }

        uint txcount = walletContract.transactionCount();
        console.log("\n>>> Initial conditions: # transaction submitted -->", txcount);
    }

    function testAttack() public {
        walletContract.getTransactionCount(true, true);
    }
}
```

```solidity
/**
 *Submitted for verification at Etherscan.io on 2021-03-04
*/

/**
 *Submitted for verification at Etherscan.io on 2018-05-10
*/

pragma solidity ^0.8.13;

/// @title Multisignature wallet - Allows multiple parties to agree on transactions before execution.
/// @author Stefan George - <stefan.george@consensys.net>
contract MultiSigWallet {

    uint constant public MAX_OWNER_COUNT = 50;

    event Confirmation(address indexed sender, uint indexed transactionId);
    event Revocation(address indexed sender, uint indexed transactionId);
    event Submission(uint indexed transactionId);
    event Execution(uint indexed transactionId);
    event ExecutionFailure(uint indexed transactionId);
    event Deposit(address indexed sender, uint value);
    event OwnerAddition(address indexed owner);
    event OwnerRemoval(address indexed owner);
    event RequirementChange(uint required);

    mapping (uint => Transaction) public transactions;
    mapping (uint => mapping (address => bool)) public confirmations;
    mapping (address => bool) public isOwner;
    address[] public owners;
    uint public required;
    uint public transactionCount;

    struct Transaction {
        address destination;
        uint value;
        bytes data;
        bool executed;
    }

    modifier onlyWallet() {
        if (msg.sender != address(this))
            revert();
        _;
    }

    modifier ownerDoesNotExist(address owner) {
        if (isOwner[owner])
            revert();
        _;
    }

    modifier ownerExists(address owner) {
        if (!isOwner[owner])
            revert();
        _;
    }

    modifier transactionExists(uint transactionId) {
        if (transactions[transactionId].destination == address(0))
            revert();
        _;
    }

    modifier confirmed(uint transactionId, address owner) {
        if (!confirmations[transactionId][owner])
            revert();
        _;
    }

    modifier notConfirmed(uint transactionId, address owner) {
        if (confirmations[transactionId][owner])
            revert();
        _;
    }

    modifier notExecuted(uint transactionId) {
        if (transactions[transactionId].executed)
            revert();
        _;
    }

    modifier notNull(address _address) {
        if (_address == address(0))
            revert();
        _;
    }

    modifier validRequirement(uint ownerCount, uint _required) {
        if (   ownerCount > MAX_OWNER_COUNT
            || _required > ownerCount
            || _required == 0
            || ownerCount == 0)
            revert();
        _;
    }

    /*
     * Public functions
     */
    /// @dev Contract constructor sets initial owners and required number of confirmations.
    /// @param _owners List of initial owners.
    /// @param _required Number of required confirmations.
    constructor(address[] memory _owners, uint _required)
        validRequirement(_owners.length, _required)
    {
        for (uint i=0; i<_owners.length; i++) {
            if (isOwner[_owners[i]] || _owners[i] == address(0))
                revert();
            isOwner[_owners[i]] = true;
        }
        owners = _owners;
        required = _required;
    }

    /// @dev Allows to add a new owner. Transaction has to be sent by wallet.
    /// @param owner Address of new owner.
    function addOwner(address owner)
        public
        onlyWallet
        ownerDoesNotExist(owner)
        notNull(owner)
        validRequirement(owners.length + 1, required)
    {
        isOwner[owner] = true;
        owners.push(owner);
        emit OwnerAddition(owner);
    }

    /// @dev Allows to remove an owner. Transaction has to be sent by wallet.
    /// @param owner Address of owner.
    function removeOwner(address owner)
        public
        onlyWallet
        ownerExists(owner)
    {
        isOwner[owner] = false;
        for (uint i=0; i<owners.length - 1; i++)
            if (owners[i] == owner) {
                owners[i] = owners[owners.length - 1];
                break;
            }
        owners.pop();
        if (required > owners.length)
            changeRequirement(owners.length);
        emit OwnerRemoval(owner);
    }

    /// @dev Allows to replace an owner with a new owner. Transaction has to be sent by wallet.
    /// @param owner Address of owner to be replaced.
    /// @param owner Address of new owner.
    function replaceOwner(address owner, address newOwner)
        public
        onlyWallet
        ownerExists(owner)
        ownerDoesNotExist(newOwner)
    {
        for (uint i=0; i<owners.length; i++)
            if (owners[i] == owner) {
                owners[i] = newOwner;
                break;
            }
        isOwner[owner] = false;
        isOwner[newOwner] = true;
        emit OwnerRemoval(owner);
        emit OwnerAddition(newOwner);
    }

    /// @dev Allows to change the number of required confirmations. Transaction has to be sent by wallet.
    /// @param _required Number of required confirmations.
    function changeRequirement(uint _required)
        public
        onlyWallet
        validRequirement(owners.length, _required)
    {
        required = _required;
        emit RequirementChange(_required);
    }

    /// @dev Allows an owner to submit and confirm a transaction.
    /// @param destination Transaction target address.
    /// @param value Transaction ether value.
    /// @param data Transaction data payload.
    function submitTransaction(address destination, uint value, bytes calldata data)
        public
        returns (uint transactionId)
    {
        transactionId = addTransaction(destination, value, data);
        confirmTransaction(transactionId);
    }

    /// @dev Allows an owner to confirm a transaction.
    /// @param transactionId Transaction ID.
    function confirmTransaction(uint transactionId)
        public
        ownerExists(msg.sender)
        transactionExists(transactionId)
        notConfirmed(transactionId, msg.sender)
    {
        confirmations[transactionId][msg.sender] = true;
        emit Confirmation(msg.sender, transactionId);
        executeTransaction(transactionId);
    }

    /// @dev Allows an owner to revoke a confirmation for a transaction.
    /// @param transactionId Transaction ID.
    function revokeConfirmation(uint transactionId)
        public
        ownerExists(msg.sender)
        confirmed(transactionId, msg.sender)
        notExecuted(transactionId)
    {
        confirmations[transactionId][msg.sender] = false;
        emit Revocation(msg.sender, transactionId);
    }

    /// @dev Allows anyone to execute a confirmed transaction.
    /// @param transactionId Transaction ID.
    function executeTransaction(uint transactionId)
        public
        notExecuted(transactionId)
    {
        if (isConfirmed(transactionId)) {
            Transaction storage txl = transactions[transactionId];
            txl.executed = true;
            (bool success,) = txl.destination.call{value:txl.value}(txl.data);
            if (success) {
                emit Execution(transactionId);
            } else {
                emit ExecutionFailure(transactionId);
                txl.executed = false;
            }
        }
    }

    /// @dev Returns the confirmation status of a transaction.
    /// @param transactionId Transaction ID.
    function isConfirmed(uint transactionId)
        public
        view
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

    /*
     * Internal functions
     */
    /// @dev Adds a new transaction to the transaction mapping, if transaction does not exist yet.
    /// @param destination Transaction target address.
    /// @param value Transaction ether value.
    /// @param data Transaction data payload.
    function addTransaction(address destination, uint value, bytes calldata data)
        internal
        notNull(destination)
        returns (uint transactionId)
    {
        transactionId = transactionCount;
        transactions[transactionId] = Transaction({
            destination: destination,
            value: value,
            data: data,
            executed: false
        });
        transactionCount += 1;
        emit Submission(transactionId);
    }

    /*
     * Web3 call functions
     */
    /// @dev Returns number of confirmations of a transaction.
    /// @param transactionId Transaction ID.
    function getConfirmationCount(uint transactionId)
        public
        view
        returns (uint count)
    {
        for (uint i=0; i<owners.length; i++)
            if (confirmations[transactionId][owners[i]])
                count += 1;
    }

    /// @dev Returns total number of transactions after filers are applied.
    /// @param pending Include pending transactions.
    /// @param executed Include executed transactions.
    function getTransactionCount(bool pending, bool executed)
        public
        view
        returns (uint count)
    {
        for (uint i=0; i<transactionCount; i++)
            if (   pending && !transactions[i].executed
                || executed && transactions[i].executed)
                count += 1;
    }

    /// @dev Returns list of owners.
    function getOwners()
        public
        view
        returns (address[] memory allo)
    {
        return owners;
    }

    /// @dev Returns array with owner addresses, which confirmed transaction.
    /// @param transactionId Transaction ID.
    function getConfirmations(uint transactionId)
        public
        view
        returns (address[] memory _confirmations)
    {
        address[] memory confirmationsTemp = new address[](owners.length);
        uint count = 0;
        uint i;
        for (i=0; i<owners.length; i++)
            if (confirmations[transactionId][owners[i]]) {
                confirmationsTemp[count] = owners[i];
                count += 1;
            }
        _confirmations = new address[](count);
        for (i=0; i<count; i++)
            _confirmations[i] = confirmationsTemp[i];
    }

    /// @dev Returns list of transaction IDs in defined range.
    /// @param from Index start position of transaction array.
    /// @param to Index end position of transaction array.
    /// @param pending Include pending transactions.
    /// @param executed Include executed transactions.
    function getTransactionIds(uint from, uint to, bool pending, bool executed)
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
        _transactionIds = new uint[](to - from);
        for (i=from; i<to; i++)
            _transactionIds[i - from] = transactionIdsTemp[i];
    }
}

```

##

I've also tested it in Remix the following way:

* Add the `MultiSigWallet.sol` to Remix
* Add the following code at the end of the Constructor

```
        // add dummy txs
        bytes memory localBytes = new bytes(1);
        for (uint k=0; k<1100; k++) {
            submitTransaction(owners[0], 0, localBytes);
        }
```

* Deploy with 2 valid owner and 2 required
* Call `getTransactionCount` with true, true
* Check the execution gas cost, it will be almost 3M, which is the limit.
* Call `submitTransaction` about 150 times to reach 1250 (valid address, 0 and 0x00)
* Call `getTransactionCount` with true, true
  * This will revert, running out of gas. While all those transaction are pending, even if you try to call it only looking for executed, it will still blow up or be very close to.
