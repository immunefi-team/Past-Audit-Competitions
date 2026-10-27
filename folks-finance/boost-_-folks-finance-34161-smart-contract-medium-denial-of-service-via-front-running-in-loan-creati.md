# Boost \_ Folks Finance 34161 - \[Smart Contract - Medium] Denial of Service via Front-Running in Loan

Submitted on Tue Aug 06 2024 03:54:20 GMT-0400 (Atlantic Standard Time) by @OxG0P1 for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34161

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

An attacker can front-run the loan creation of other users by inputting the same `loanId`, causing a Denial of Service (DoS) for the affected users.

## Vulnerability Details

A user can create a loan by specifying a `loanId` of their choice.

```solidity
function createLoanAndDeposit(
    Messages.MessageParams memory params,
    bytes32 accountId,
    bytes32 loanId,
    uint256 amount,
    uint16 loanTypeId,
    bytes32 loanName
) external payable nonReentrant {
    _doOperation(
        params,
        Messages.Action.CreateLoanAndDeposit,
        accountId,
        amount,
        abi.encodePacked(loanId, poolId, amount, loanTypeId, loanName)
    );
}
```

In `LoanManager.sol`, the function first checks whether there is an active loan associated with the specified `loanId`. If no active loan exists, the loan will be created with the provided `loanId`. However, if an active loan with the same `loanId` already exists, the transaction will revert.

```solidity
if (isUserLoanActive(loanId)) revert UserLoanAlreadyCreated(loanId);
```

```solidity
function isUserLoanActive(bytes32 loanId) public view returns (bool) {
    return _userLoans[loanId].isActive;
}
```

This can be problematic because an attacker can front-run the loan creation transaction from the Hub chain, leading to a DoS for the victim.

## Impact Details

An attacker can cause a DoS for all loan creation transactions of other users. Since loan management is conducted on the Hub chain, a user who wishes to create a loan from another spoke chain must relay the message through a bridge. An attacker can front-run this transaction, causing the victim's transaction to revert. This results in a loss for the victim, as they still incur the bridge fee required for relaying the message.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/LoanManager.sol#L49>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/LoanManagerState.sol#L413-L415>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/spoke/SpokeToken.sol#L46-L61>

## Proof of concept

## Proof of Concept

Consider the following scenario with two users, Alice (the victim) and Bob (the attacker):

1. **Alice's Loan Creation Intent**:

   * Alice intends to create a loan with the `loanId` of `0x123456`.

   ```solidity
   createLoanAndDeposit(params, accountIdAlice, 0x123456, amount, loanTypeId, loanName);
   ```
2. **Bob Monitors and Front-Runs**:

   * Bob, monitoring the mempool, detects Alice's intent to create a loan with `loanId` `0x123456`.
   * Bob immediately sends a transaction with the same `loanId`:

   ```solidity
   createLoanAndDeposit(params, accountIdBob, 0x123456, amount, loanTypeId, loanName);
   ```
3. **Transaction Processing**:

   * Bob's transaction is processed first due to higher gas fees, creating a loan with `loanId` `0x123456`.

   ```solidity
   // In LoanManager.sol
   if (isUserLoanActive(loanId)) revert UserLoanAlreadyCreated(loanId);
   ```

   * The `isUserLoanActive` function checks if a loan with `loanId` `0x123456` is active. Since Bob's loan is now active, the function returns `true`.
4. **Alice's Transaction Fails**:
   * When Alice's transaction is processed, the `isUserLoanActive` function again checks the `loanId` `0x123456`. It finds an active loan created by Bob, causing Alice's transaction to revert with `UserLoanAlreadyCreated(loanId)`.
5. **Result**:
   * Alice's transaction fails, and she loses the gas fee for the transaction. If Alice's transaction was relayed from another chain via a bridge, she also loses the bridge fee paid for relaying the message.
