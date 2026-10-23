# Boost \_ Folks Finance 33611 - \[Smart Contract - Medium] Adversary can perform a DoS on users createL

Submitted on Wed Jul 24 2024 14:23:04 GMT-0400 (Atlantic Standard Time) by @nnez for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33611

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Description

A loan position is represented by a loan struct, each with a unique 32-byte identifier, `loanId`. Before using the protocol's lending/borrowing features, users must create a loan. Users can freely choose their `loanId` if it is not already taken by other users.

Two actions are used to create loan, `createLoan` and `createLoanAndDeposit` and users can perform these actions via `SpokeCommon` and `SpokeToken` endpoint.\
Here is the execution flow when users try to perform these actions on Spoke Chain.

Let's say Alice initiates a transaction by calling `createLoan` or `createLoanAndDeposit` from Spoke Chain

```
-- Spoke Chain --
--> SpokeCommon/SpokeToken#createLoan --> Wormhole Network
-- Hub Chain --
--> Wormhole Relayer --> Wormhole Adapter --> BridgeRouter --> createLoan on Hub
```

There will be a delay before the transaction reaches the Hub Chain and is executed on the `BridgeRouter` contract. If someone else creates a loan with the same `loanId` as Alice's, her transaction will revert and be stored in the `failedMessages` list.

Although there is a `retryMessage` function as a fallback, Alice's transaction will always revert because the `loanId` is already taken by someone else.\
Alice would have no choice but to `reverseMessage` in the case that she initiated the transaction using `createLoanAndDeposit`, in order to get her token back.

This opens an attack vector for adversaries to DoS other users, preventing them from performing `createLoan` and `createLoanAndDeposit` and using the protocol's lending/borrowing features.

### Attack Scenario

1. Alice initiates a transaction on Spoke Chain to perform `createLoanAndDeposit` on Hub Chain
2. Eve notices Alice's transaction and proceed to perform `createLoanAndDeposit` with Alice's intended `loanId` on SpokeToken endpoint on Hub Chain (No delay)
3. When Alice's transaction arrives on Hub Chain, it will revert.

### Related code snippets

```solidity
File: /contracts/hub/LoanManager.sol
function createUserLoan(
    bytes32 loanId,
    bytes32 accountId,
    uint16 loanTypeId,
    bytes32 loanName
) external override onlyRole(HUB_ROLE) nonReentrant {
    // check loan types exists, is not deprecated and no existing user loan for same loan id
    if (!isLoanTypeCreated(loanTypeId)) revert LoanTypeUnknown(loanTypeId);
    if (isLoanTypeDeprecated(loanTypeId)) revert LoanTypeDeprecated(loanTypeId);
    if (isUserLoanActive(loanId)) revert UserLoanAlreadyCreated(loanId); @<-- The loan creation will revert if the loanId is already taken

    // create loan
    UserLoan storage userLoan = _userLoans[loanId];
    userLoan.isActive = true;
    ...
    ... snipped
    ...

File: /contracts/hub/LoanManagerState.sol
function isUserLoanActive(bytes32 loanId) public view returns (bool) {
    return _userLoans[loanId].isActive;
}
```

## Impact

* Griefing, preventing users to execute `createLoan` and `createLoanAndDeposit` function. In the case of `createLoanAndDeposit`, users are forced to reverse message back, thus, costing them the cross-chain messaging fee.

## Recommend Mitigation

* Consider incorporating user's specific unique identifier in `loanId`. I'd suggest incorporating `msg.sender` in `loanId`, this leaves ample room for loan number (12 bytes).

## Proof of concept

## Proof-of-Concept

A test file in the secret gist demonstrates the following:

* `createLoanAndDeposit` will fail if `loanId` is already taken by attacker.
* `retryMessage` on that transaction will also always revert.

**Steps**

1. Run `forge init --no-commit --no-git --vscode`.
2. Create a new test file, `FolksGrief.t.sol` in `test` directory.
3. Put the test from secret gist in the file: <https://gist.github.com/nnez/48483c82fd3645bd82c0f778a87b97bf>
4. Run `forge t --match-contract FolksGrief -vvvv`
5. Observe in two events `MessageFailed` and `MessageRetryFailed` emitted in call traces and the reason starts with `0xdcbc8448` which corresponds with `UserLoanAlreadyCreated` error.
