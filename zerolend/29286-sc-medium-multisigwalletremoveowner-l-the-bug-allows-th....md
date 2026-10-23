# 29286 - \[SC - Medium] MultiSigWalletremoveOwner - L The bug allows th...

## `MultiSigWallet::removeOwner()` - L127: The bug allows the last owner in the `owners` array to remain in the array even after being marked as not an owner if they are the one intended for removal.

Submitted on Mar 13th 2024 at 09:20:08 UTC by @OxSCSamurai for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29286

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/zerolend/governance>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

### Description

### Brief/Intro

`MultiSigWallet::removeOwner()` - L127: The bug allows the last owner in the `owners` array to remain in the array even after being marked as not an owner if they are the one intended for removal.

* Although this is a serious bug, it's not obvious to me which impact in scope fits best here. Again, I believe the available impacts in scope are limiting and could compromise protocol security. This bug report is one example of a very serious bug which doesn't seem to have an obvious impact in scope match. This is something for Immunefi team to look at and try improve, because I could have easily decided submitting this bug report isn't worth my time and effort. But I know better and helping secure web3 is top priority, therefore I've pulled all the strings in an effort to make at least one impact in scope fit well enough.

> But alas, heed my advice, take this bug report seriously, as the bug has a tendency to mutate into a critical level monster over time...

### Vulnerability Details

## SUMMARY:

The buggy function, slightly modified so that I didnt need to deal with `onlyWallet` access control during tests:

```solidity
    /// @dev Allows to remove an owner. Transaction has to be sent by wallet.
    /// @param owner Address of owner.
    //function removeOwner(address owner) public onlyWallet ownerExists(owner) {
    function removeOwner(address owner) public ownerExists(owner) { 
        isOwner[owner] = false;
        for (uint256 i = 0; i < owners.length - 1; i++)
            if (owners[i] == owner) {
                owners[i] = owners[owners.length - 1];
                owners.pop();
                break;
            }
        if (required > owners.length) changeRequirement(owners.length);
        emit OwnerRemoval(owner);
    }
```

The bug in the `removeOwner()` function of the ZeroLend protocol's multisignature wallet contract allows the last owner in the `owners` array to remain in the array even after being marked as not an owner, if they are the one intended for removal. This occurs because the loop condition in the function stops iterating before reaching the last owner, resulting in their continued & permanent presence in the array.

The bug in this function lies within the loop condition:

```solidity
for (uint256 i = 0; i < owners.length - 1; i++)
```

This loop will iterate over all owners except the last one due to the combo of `i <` and `owners.length - 1`. If the owner to be removed happens to be the last one in the `owners` array, this loop will not process it, resulting in the owner not being removed from the array.

## NOTE:

* Regardless of whether this bug poses an immediate threat/risk to the protocol or not, it is almost certainly guaranteed to pose a critical level risk eventually if the bug is not fixed, simply because each time an owner is removed who happened to be in the last position in the `owners` array, the problem compounds.
* In other words, the longer the bug is left unfixed the worse the problem will get over time until it becomes a critical level problem where the `owners` array usage is DoS'ed along with ability to confirm transactions, which is one of the core functionalities of the multisig. The ability to confirm transactions will be permanently DoS'ed...

> That's it, PoC done, critical bug proven, can I have my pizza now please? 👀

> Not so fast Samurai! ⚔️

### Impact Details

## POTENTIAL IMPACTS IN SCOPE:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
* Temporary freezing of funds for at least 1 hour
* ?

## IMPACTS:

* the immediate and obvious impact is that any owner to be removed that's in the last position of the `owners` array will be permanently stuck in the array, even when it's `isOwner` mapping will be set to `false` successfully, so he wont be able to take any governance/signer actions at least, but can NEVER remove this owner from the array.
* will be able to add incompletely removed owner back to the `owners` array but as a separate & duplicate array entry only
* can affect accuracy of results coming from `getConfirmationCount()`
* over time it's possible(almost guaranteed) for `required` to become greater than the available usable owners in the `owners` array

Some possible worst case scenario impacts:

* it's possible for same owner/signer to double vote/confirm, and is enabled by: unsuccessful removal + add same owner back again, resulting in both entries in `owners` array linked to the same `isOwner` mapping, so to speak. So whenever there's a for loop looping through the `owners` array, this owner's contribution(vote/confirmation) will be double-counted/double-processed, which could be tantamount to unintentional governance vote result manipulation.
* DoS of owners array usability and therefore DoS of ability to vote/confirm transactions: what makes this possible is when all owners have been removed when each of them were the last entry in the `owners` array, and this continued until the `owners` array reached a point where the max limit of 50 owners/confirmations(I need to confirm this?) were reached, then its impossible to add new owners or replace existing owners, while all existing owners are unusable and not replaceable, resulting in permanent DoS of critical multisig functionality.

### References

<https://github.com/zerolend/governance/blob/84d48522cdb14f4a5a4aefc4059c0ea1e2e97afa/contracts/governance/MultiSigWallet.sol#L127>

### Proof of Concept

## PROOF OF CONCEPT (PoC):

TEST 1: Proving that the bug exists, and that my bugfix works TEST 2: Proving that the unsuccessfully removed owner can never be fully removed ever again, nor replaced TEST 3: Proving it's possible for same owner to get duplicated in `owners` array, enabling double voting/confirmation counts TEST 4: Proving that the accuracy of results from `isConfirmed()` and `getConfirmationCount()` can be affected

These two tests I will do PoC for later, will add to this bug report once ready: TEST 5: Proving it's possible to permanently DoS `owners` array and ability to vote/confirm transactions TEST 6: Proving it's possible for `required` to become greater than the number of usable owners/signers

TESTS:

## TEST 1: Proving that the bug exists, and that my bugfix works

Foundry based test contract used for this test:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.23 <0.9.0;

import "forge-std/Test.sol";

import "governance/contracts/governance/MultiSigWallet.sol";

contract TestMultisigRemoveOwner is Test {
    MultiSigWallet multiSigWallet;

    function setUp() public {
        address[] memory initialOwners = new address[](3);
        // Populate initialOwners with appropriate addresses
        initialOwners[0] = address(0x1234567890123456789012345678901234567890);
        initialOwners[1] = address(0x5234567890123456789012345678901234567899);
        initialOwners[2] = address(0x9999999999999999999999999999999999999999);

        uint256 requiredConfirmations = 2;

        // Deploy the MultiSigWallet contract
        multiSigWallet = new MultiSigWallet(initialOwners, requiredConfirmations);
    }

    function test_removeOwner() external {

        address[] memory _owners = multiSigWallet.getOwners();
        // Try to remove the last owner in the array
        multiSigWallet.removeOwner(_owners[_owners.length - 1]);

        // If not removed, array length should still be 3
        assert(multiSigWallet.getOwners().length == 3); // if this passes, then owner in last position of `owners` array, was not removed.
        // If removed, array length should be 2
        //assert(multiSigWallet.getOwners().length == 2); // if this passes, then owner in last position of `owners` array, was successfully removed.

    }
}
```

Note: obviously alternate between commenting out the relevant assert statements above when running the tests for the bug unfixed and bug fixed.

## Test results: bug not fixed: `for (uint256 i = 0; i < owners.length - 1; i++)`

Using test function:

```solidity
    function test_removeOwner() external {

        address[] memory _owners = multiSigWallet.getOwners();
        // Try to remove the last owner in the array
        multiSigWallet.removeOwner(_owners[_owners.length - 1]);

        // If not removed, array length should still be 3
        assert(multiSigWallet.getOwners().length == 3); // if this passes, then owner in last position of `owners` array, was not removed.
        // If removed, array length should be 2
        //assert(multiSigWallet.getOwners().length == 2); // if this passes, then owner in last position of `owners` array, was successfully removed.
    }
```

Test result:

```solidity
$ forge test --contracts test/TestMultisigRemoveOwner.t.sol --mt test_removeOwner -vvvvv
[⠒] Compiling...
[⠢] Compiling 1 files with 0.8.23
[⠆] Solc 0.8.23 finished in 1.96s
Compiler run successful!

Ran 1 test for test/TestMultisigRemoveOwner.t.sol:TestMultisigRemoveOwner
[PASS] test_removeOwner() (gas: 24758)
Traces:
  [1499074] TestMultisigRemoveOwner::setUp()
    ├─ [1442244] → new MultiSigWallet@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← 6302 bytes of code
    └─ ← ()

  [24758] TestMultisigRemoveOwner::test_removeOwner()
    ├─ [9566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [10405] MultiSigWallet::removeOwner(0x9999999999999999999999999999999999999999)
    │   ├─ emit OwnerRemoval(owner: 0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [1566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.45ms (533.74µs CPU time)

Ran 1 test suite in 1.12s (2.45ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

It's clear from the result that the removal was unsuccessful, but the function was still successfully executed as can be seen by the emitted event `OwnerRemoval()`, which also indicates that `isOwner` was successfully updated to `false`, making it impossible to try remove this same owner again, effectively leaving this owner permanently stuck in the `owners` array.

## Test results: bug fixed: `for (uint256 i = 0; i < owners.length; i++)`

Using test function:

```solidity
    function test_removeOwner() external {

        address[] memory _owners = multiSigWallet.getOwners();
        // Try to remove the last owner in the array
        multiSigWallet.removeOwner(_owners[_owners.length - 1]);

        // If not removed, array length should still be 3
        //assert(multiSigWallet.getOwners().length == 3); // if this passes, then owner in last position of `owners` array, was not removed.
        // If removed, array length should be 2
        assert(multiSigWallet.getOwners().length == 2); // if this passes, then owner in last position of `owners` array, was successfully removed.
    }
```

Test result:

```solidity
$ forge test --contracts test/TestMultisigRemoveOwner.t.sol --mt test_removeOwner -vvvvv
[⠒] Compiling...
[⠢] Compiling 2 files with 0.8.23
[⠆] Solc 0.8.23 finished in 2.02s
Compiler run successful!

Ran 1 test for test/TestMultisigRemoveOwner.t.sol:TestMultisigRemoveOwner
[PASS] test_removeOwner() (gas: 26743)
Traces:
  [1496667] TestMultisigRemoveOwner::setUp()
    ├─ [1439844] → new MultiSigWallet@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← 6290 bytes of code
    └─ ← ()

  [29075] TestMultisigRemoveOwner::test_removeOwner()
    ├─ [9566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [17622] MultiSigWallet::removeOwner(0x9999999999999999999999999999999999999999)
    │   ├─ emit OwnerRemoval(owner: 0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [1292] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899]
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 881.56µs (179.17µs CPU time)

Ran 1 test suite in 1.49s (881.56µs CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

It's clear from the test result that my bugfix worked and now the same owner is removed successfully from the `owners` array.

## TEST 2: Proving that the unsuccessfully removed owner can never be fully removed ever again, nor replaced

Since both the successful and unsuccessful(due to bug) removal of an owner from the `owners` array sets the `isOwner` mapping of the owner to `false`, there is no way of successfully calling the `removeOwner()` function again for same owner, as the `ownerExists(owner)` modifier will trigger a revert.

## Modified test function used to try remove same owner again:

```solidity
    function test_removeOwner() external {

        address[] memory _owners = multiSigWallet.getOwners();
        // Try to remove the last owner in the array
        multiSigWallet.removeOwner(_owners[_owners.length - 1]);

        // If not removed, array length should still be 3
        assert(multiSigWallet.getOwners().length == 3); // if this passes, then owner in last position of `owners` array, was not removed.
        // If removed, array length should be 2
        //assert(multiSigWallet.getOwners().length == 2); // if this passes, then owner in last position of `owners` array, was successfully removed.

		// After above unsuccessful removal, try to call remove function again for same owner. Should revert via modifier.
        _owners = multiSigWallet.getOwners();
        vm.expectRevert();
        multiSigWallet.removeOwner(_owners[_owners.length - 1]);
    }
```

## Test result:

```solidity
Traces:
  [1499074] TestMultisigRemoveOwner::setUp()
    ├─ [1442244] → new MultiSigWallet@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← 6302 bytes of code
    └─ ← ()

  [31702] TestMultisigRemoveOwner::test_removeOwner()
    ├─ [9566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [10405] MultiSigWallet::removeOwner(0x9999999999999999999999999999999999999999)
    │   ├─ emit OwnerRemoval(owner: 0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [1566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [1566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [0] VM::expectRevert(custom error f4844814:)
    │   └─ ← ()
    ├─ [560] MultiSigWallet::removeOwner(0x9999999999999999999999999999999999999999)
    │   └─ ← EvmError: Revert
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.18ms (260.14µs CPU time)

Ran 1 test suite in 1.29s (1.18ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

As expected the function reverted via the modifier's check. Not possible to call the remove function again for same owner after owner was successfully or unsuccessfully removed.

## Now lets check if this same unsuccessfully removed owner can be replaced instead via the `replaceOwner()` function:

Using the following modified test functions:

```solidity
    function test_removeOwner() external {

        address[] memory _owners = multiSigWallet.getOwners();
        // Try to remove the last owner in the array
        multiSigWallet.removeOwner(_owners[_owners.length - 1]);

        // If not removed, array length should still be 3
        assert(multiSigWallet.getOwners().length == 3); // if this passes, then owner in last position of `owners` array, was not removed.
        // If removed, array length should be 2
        //assert(multiSigWallet.getOwners().length == 2); // if this passes, then owner in last position of `owners` array, was successfully removed.

        _owners = multiSigWallet.getOwners();
        vm.expectRevert();
        multiSigWallet.removeOwner(_owners[_owners.length - 1]);

        // Now try to replace this same owner that was unsuccessfully removed so far:
        test_replaceOwner();
    }

    function test_replaceOwner() internal {
        address[] memory _owners = multiSigWallet.getOwners();
        // should revert because the failed owner removal from above test will also fail the `ownerExists` modifier check
        vm.expectRevert();
        multiSigWallet.replaceOwner(_owners[_owners.length - 1], address(0x8234567890123456789012345678901234567899));
        assert(multiSigWallet.getOwners()[_owners.length - 1] == address(0x9999999999999999999999999999999999999999));
    }
```

## Test result:

```solidity
  [39349] TestMultisigRemoveOwner::test_removeOwner()
    ├─ [9566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [10405] MultiSigWallet::removeOwner(0x9999999999999999999999999999999999999999)
    │   ├─ emit OwnerRemoval(owner: 0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [1566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [1566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [0] VM::expectRevert(custom error f4844814:)
    │   └─ ← ()
    ├─ [560] MultiSigWallet::removeOwner(0x9999999999999999999999999999999999999999)
    │   └─ ← EvmError: Revert
    ├─ [1566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [0] VM::expectRevert(custom error f4844814:)
    │   └─ ← ()
    ├─ [726] MultiSigWallet::replaceOwner(0x9999999999999999999999999999999999999999, 0x8234567890123456789012345678901234567899)
    │   └─ ← EvmError: Revert
    ├─ [1566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.32ms (519.20µs CPU time)

Ran 1 test suite in 1.04s (1.32ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

`replaceOwner()` reverted as expected.

## TEST 3: Proving it's possible for same owner to get duplicated in `owners` array, enabling double voting/confirmation counts

* There's a way to double vote, and thats by removing an owner/signer who's at the end of the owners array, and then adding them back again. This will cause two separate entries for same owner address into the owners array, both linked/mapped to the isOwner mapping, thereby making it totally possible for this owner to double vote when the for loop loops through the owners list. STEPS:
* remove owner in last array position
* add owner back

Modified test function used:

```solidity
    function test_removeOwner() external {

        address[] memory _owners = multiSigWallet.getOwners();
        // Try to remove the last owner in the array
        multiSigWallet.removeOwner(_owners[_owners.length - 1]);

        // If not removed, array length should still be 3
        assert(multiSigWallet.getOwners().length == 3); // if this passes, then owner in last position of `owners` array, was not removed.

        // Now to test owner duplication in `owners` array, including double voting/confirmation:
        // After above failed remove need to add same owner back to `owners` array
        multiSigWallet.addOwner(address(0x9999999999999999999999999999999999999999));
        _owners = multiSigWallet.getOwners();
        // Should now have same owner in last two positions of array, asserting that owner was added successfully after partial/unsuccessful removal
        // Last array position:
        assert(multiSigWallet.getOwners()[_owners.length - 1] == address(0x9999999999999999999999999999999999999999));
        // Second last array position:
        assert(multiSigWallet.getOwners()[_owners.length - 2] == address(0x9999999999999999999999999999999999999999));

    }
```

## Test result:

```solidity
  [65104] TestMultisigRemoveOwner::test_removeOwner()
    ├─ [9566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [10405] MultiSigWallet::removeOwner(0x9999999999999999999999999999999999999999)
    │   ├─ emit OwnerRemoval(owner: 0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [1566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [27640] MultiSigWallet::addOwner(0x9999999999999999999999999999999999999999)
    │   ├─ emit OwnerAddition(owner: 0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [1840] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999, 0x9999999999999999999999999999999999999999]
    ├─ [1840] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999, 0x9999999999999999999999999999999999999999]
    ├─ [1840] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999, 0x9999999999999999999999999999999999999999]
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.76ms (681.45µs CPU time)

Ran 1 test suite in 1.26s (2.76ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Above test result proves it's possible to add an owner twice to the `owners` array, as can be seen by the last two entries of the array. During for loops when the `owners` array is looped through, this owner will be counted twice, which could potentially lead to double voting/double confirmations.

## TEST 4: Proving that the accuracy of results from `isConfirmed()` and `getConfirmationCount()` can be affected

* when an owner is unsuccessfully removed and then added again afterwards, his single transaction confirmation will be counted twice instead of just once, which is equivalent to voting twice instead of just once, or signing a tx twice...

Test contract was modified for this one, as per below, with `requiredConfirmations = 1`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.23 <0.9.0;

import "forge-std/Test.sol";

import "governance/contracts/governance/MultiSigWallet.sol";


contract TestMultisigRemoveOwner is Test {
    MultiSigWallet multiSigWallet;

    function setUp() public {
        address[] memory initialOwners = new address[](3);
        // Populate initialOwners with appropriate addresses
        initialOwners[0] = address(0x1234567890123456789012345678901234567890);
        initialOwners[1] = address(0x5234567890123456789012345678901234567899);
        initialOwners[2] = address(0x9999999999999999999999999999999999999999);

        uint256 requiredConfirmations = 1;

        // Deploy the MultiSigWallet contract
        multiSigWallet = new MultiSigWallet(initialOwners, requiredConfirmations);

        vm.deal(address(multiSigWallet), 100);
    }

    function test_removeOwner() external {

        address[] memory _owners = multiSigWallet.getOwners();

        vm.startPrank(address(0x9999999999999999999999999999999999999999));
        uint256 _transactionId = multiSigWallet.submitTransaction(address(0x2934567890123456789012345678901234567885), 10, "0x");
        vm.stopPrank();
        assert(multiSigWallet.isConfirmed(_transactionId) == true);
        assert(multiSigWallet.getConfirmationCount(_transactionId) == 1);

    }

    receive() external payable {
    }
}
```

## Control test: the owner has NOT been removed yet, so all is good and NORMAL:

```solidity
  [212687] TestMultisigRemoveOwner::test_removeOwner()
    ├─ [9566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [0] VM::startPrank(0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [186489] MultiSigWallet::submitTransaction(0x2934567890123456789012345678901234567885, 10, 0x3078)
    │   ├─ emit Submission(transactionId: 0)
    │   ├─ emit Confirmation(sender: 0x9999999999999999999999999999999999999999, transactionId: 0)
    │   ├─ [0] 0x2934567890123456789012345678901234567885::fallback{value: 10}(0x3078)
    │   │   └─ ← ()
    │   ├─ emit Execution(transactionId: 0)
    │   └─ ← 0
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [3013] MultiSigWallet::isConfirmed(0) [staticcall]
    │   └─ ← true
    ├─ [2795] MultiSigWallet::getConfirmationCount(0) [staticcall]
    │   └─ ← 1
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.25ms (344.73µs CPU time)

Ran 1 test suite in 1.10s (1.25ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

For the above control test result, its clear that only one confirmation count was produced, which is correct because only one owner confirmed transaction and this owner has not been removed from the `owners` array yet.

Next is the actual test to prove that it's possible for this same owner to provide TWO confirmations instead of just one, via a single confirmation function call for the same transaction, AFTER this owner was unsuccessfully removed and added back to the `owners` array again. For this next test I will change the `requiredConfirmations = 2`, and I've again modified the test contract for this next test:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.23 <0.9.0;

import "forge-std/Test.sol";

import "governance/contracts/governance/MultiSigWallet.sol";

contract TestMultisigRemoveOwner is Test {
    MultiSigWallet multiSigWallet;

    function setUp() public {
        address[] memory initialOwners = new address[](3);
        // Populate initialOwners with appropriate addresses
        initialOwners[0] = address(0x1234567890123456789012345678901234567890);
        initialOwners[1] = address(0x5234567890123456789012345678901234567899);
        initialOwners[2] = address(0x9999999999999999999999999999999999999999);

        uint256 requiredConfirmations = 2;

        // Deploy the MultiSigWallet contract
        multiSigWallet = new MultiSigWallet(initialOwners, requiredConfirmations);

        vm.deal(address(multiSigWallet), 100);
    }

    function test_removeOwner() external {

        address[] memory _owners = multiSigWallet.getOwners();
        // Try to remove the last owner in the array
        multiSigWallet.removeOwner(_owners[_owners.length - 1]);

        // After above failed remove need to add same owner back to `owners` array
        multiSigWallet.addOwner(address(0x9999999999999999999999999999999999999999));
        _owners = multiSigWallet.getOwners();

        vm.startPrank(address(0x9999999999999999999999999999999999999999));
        uint256 _transactionId = multiSigWallet.submitTransaction(address(0x2934567890123456789012345678901234567885), 10, "0x");
        vm.stopPrank();
        assert(multiSigWallet.isConfirmed(_transactionId) == true);
        assert(multiSigWallet.getConfirmationCount(_transactionId) == 2);

    }

    receive() external payable {
    }
}
```

Test result:

```solidity
  [251044] TestMultisigRemoveOwner::test_removeOwner()
    ├─ [9566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [10405] MultiSigWallet::removeOwner(0x9999999999999999999999999999999999999999)
    │   ├─ emit OwnerRemoval(owner: 0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [27640] MultiSigWallet::addOwner(0x9999999999999999999999999999999999999999)
    │   ├─ emit OwnerAddition(owner: 0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [1840] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999, 0x9999999999999999999999999999999999999999]
    ├─ [0] VM::startPrank(0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [183427] MultiSigWallet::submitTransaction(0x2934567890123456789012345678901234567885, 10, 0x3078)
    │   ├─ emit Submission(transactionId: 0)
    │   ├─ emit Confirmation(sender: 0x9999999999999999999999999999999999999999, transactionId: 0)
    │   ├─ [0] 0x2934567890123456789012345678901234567885::fallback{value: 10}(0x3078)
    │   │   └─ ← ()
    │   ├─ emit Execution(transactionId: 0)
    │   └─ ← 0
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [3951] MultiSigWallet::isConfirmed(0) [staticcall]
    │   └─ ← true
    ├─ [3610] MultiSigWallet::getConfirmationCount(0) [staticcall]
    │   └─ ← 2
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.36ms (399.42µs CPU time)

Ran 1 test suite in 1.34s (1.36ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

It's clear from the above test result that this owner was able to unintentionally double confirm the same transaction, effectively double voting or double signing, which would obviously adversely affect governance voting outcomes.

## Bonus test: activate my bugfix for exactly the same test conditions as above test, lets see if owner can still double confirm!

```solidity
  [164875] TestMultisigRemoveOwner::test_removeOwner()
    ├─ [9566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [17622] MultiSigWallet::removeOwner(0x9999999999999999999999999999999999999999)
    │   ├─ emit OwnerRemoval(owner: 0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [2940] MultiSigWallet::addOwner(0x9999999999999999999999999999999999999999)
    │   ├─ emit OwnerAddition(owner: 0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [1566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [0] VM::startPrank(0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [125633] MultiSigWallet::submitTransaction(0x2934567890123456789012345678901234567885, 10, 0x3078)
    │   ├─ emit Submission(transactionId: 0)
    │   ├─ emit Confirmation(sender: 0x9999999999999999999999999999999999999999, transactionId: 0)
    │   └─ ← 0
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [3158] MultiSigWallet::isConfirmed(0) [staticcall]
    │   └─ ← false
    └─ ← panic: assertion failed (0x01)

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 2.17ms (517.29µs CPU time)

Ran 1 test suite in 1.55s (2.17ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/TestMultisigRemoveOwner.t.sol:TestMultisigRemoveOwner
[FAIL. Reason: panic: assertion failed (0x01)] test_removeOwner() (gas: 173275)

Encountered a total of 1 failing tests, 0 tests succeeded
```

The `requiredConfirmations = 2` in above bonus test, but due to my bugfix the owner could not double confirm, could only confirm once, that's why the assertion failed. And if I change it to `requiredConfirmations = 1` and rerun with my bugfix, we get the following successful result:

```solidity
  [224838] TestMultisigRemoveOwner::test_removeOwner()
    ├─ [9566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [17622] MultiSigWallet::removeOwner(0x9999999999999999999999999999999999999999)
    │   ├─ emit OwnerRemoval(owner: 0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [2940] MultiSigWallet::addOwner(0x9999999999999999999999999999999999999999)
    │   ├─ emit OwnerAddition(owner: 0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [1566] MultiSigWallet::getOwners() [staticcall]
    │   └─ ← [0x1234567890123456789012345678901234567890, 0x5234567890123456789012345678901234567899, 0x9999999999999999999999999999999999999999]
    ├─ [0] VM::startPrank(0x9999999999999999999999999999999999999999)
    │   └─ ← ()
    ├─ [182489] MultiSigWallet::submitTransaction(0x2934567890123456789012345678901234567885, 10, 0x3078)
    │   ├─ emit Submission(transactionId: 0)
    │   ├─ emit Confirmation(sender: 0x9999999999999999999999999999999999999999, transactionId: 0)
    │   ├─ [0] 0x2934567890123456789012345678901234567885::fallback{value: 10}(0x3078)
    │   │   └─ ← ()
    │   ├─ emit Execution(transactionId: 0)
    │   └─ ← 0
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [3013] MultiSigWallet::isConfirmed(0) [staticcall]
    │   └─ ← true
    ├─ [2795] MultiSigWallet::getConfirmationCount(0) [staticcall]
    │   └─ ← 1
    └─ ← ()

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.54ms (485.38µs CPU time)

Ran 1 test suite in 1.60s (1.54ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

With my bugfix the owner was removed successfully, added back to `owners` array again, confirmed a transaction successfully only once, and all good as it should be.

## TEST 5: Proving it's possible to permanently DoS `owners` array and ability to vote/confirm transactions

* Will add this test later if I manage to do it and it's successful.

## TEST 6: Proving it's possible for `required` to become greater than the number of usable owners/signers

* Will add this test later if I manage to do it and it's successful.

## BUGFIX:

To fix this, you should use `owners.length` instead of `owners.length - 1` as the loop condition:

```solidity
for (uint256 i = 0; i < owners.length; i++)
```

With this change, the loop will iterate over all elements in the `owners` array, ensuring that the owner to be removed is correctly handled regardless of its position in the array.
