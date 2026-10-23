# Boost \_ Folks Finance 33893 - \[Smart Contract - Medium] Malicious users can DoS loan creations and d

Submitted on Thu Aug 01 2024 07:57:41 GMT-0400 (Atlantic Standard Time) by @iamandreiski for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33893

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Since both accountIds and loanIds are created by the users (as opposed to a hash from different parameters), when a user initiates a loan creation and deposit into one transaction, that deposit can be DoSd by frontrunning it with a loan creation transaction with the same ID. Since the user has also deposited (besides initiating a loan creation), they would then have to "reverse" the message in order to get their money back, which would cost additional funds in terms of gas costs, besides temporarily freezing their funds. This "attack" can be repeated an indefinite amount of times. Since all checks on whether an account exists happen on the Hub chain, a user's message won't fail until it has reached the Hub chain.

## Vulnerability Details

When a user wants to create a loan and deposit in the same transaction, the user path which they can take is through the Spoke chain's token contract calling the `createLoanAndDeposit` function:

```
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

The problem is that the `loanId` argument is determined by the user and can be arbitrary. Since loans with "same IDs" can't exist, the problem arises if a malicious user frontruns the `createLoanAndDeposit` transaction with a `createLoan` transaction with the same loan ID.

Since the checks whether the loan exists or not are performed on the Hub chain, once the transaction is initiated on the spoke chain, it will be relayed through the desired adapter (CCIP, Wormhole).

Once the data/token transfer is relayed to the Hub chain, the `_receiveMessage` function on Hub.sol will be invoked with the `Message.Action.CreateLoanAndDeposit`:

```
 else if (payload.action == Messages.Action.CreateLoanAndDeposit) {
            bytes32 loanId = payload.data.toBytes32(index);
            index += 32;
            uint8 poolId = payload.data.toUint8(index);
            index += 1;
            uint256 amount = payload.data.toUint256(index);
            index += 32;
            uint16 loanTypeId = payload.data.toUint16(index);
            index += 2;
            bytes32 loanName = payload.data.toBytes32(index);

            loanManager.createUserLoan(loanId, payload.accountId, loanTypeId, loanName);
            loanManager.deposit(loanId, payload.accountId, poolId, amount);

            // save token received
            receiveToken = ReceiveToken({ poolId: poolId, amount: amount });
```

When `loanManager.createUserLoan` is first called, the transaction will revert due to the following check:

```
 function createUserLoan(
        bytes32 loanId,
        bytes32 accountId,
        uint16 loanTypeId,
        bytes32 loanName
    ) external override onlyRole(HUB_ROLE) nonReentrant {
        // check loan types exists, is not deprecated and no existing user loan for same loan id
        if (!isLoanTypeCreated(loanTypeId)) revert LoanTypeUnknown(loanTypeId);
        if (isLoanTypeDeprecated(loanTypeId)) revert LoanTypeDeprecated(loanTypeId);
        if (isUserLoanActive(loanId)) revert UserLoanAlreadyCreated(loanId);
```

Since the transaction was frontrunned by the malicious user, and such a loanId already exists, this action will fail and store the message as a failed one due to the following check whether the loan is active:

```
    function isUserLoanActive(bytes32 loanId) public view returns (bool) {
        return _userLoans[loanId].isActive;
    }

```

Since this message is non-executable (it will keep failing due to the loanId being active), as explained above, and as it will always revert, the only plausible solution is for the user to reverse their message and get back the funds.

This is an action that will cost additional funds in terms of gas + the temporary freezing of funds until the message is reversed and the funds relayed back.

## Impact Details

All `createLoanAndDeposit` transactions can be DoSd by frontrunning them with a loan creation transaction with the same loan id as the unsuspecting user(victim) who wants to create it. Once this is acheived, the user whose transaction was DoSd on the destination chain (HUB) would have to reverse their message in order to get their funds back. This will cause additional gas costs, as well as a temporary freezing of funds until they're relayed back. The attack can be repeated an indefinite amount of times.

## References

PoC uses Foundry and the MockLoanManager contract was modified to include a mapping called \_userLoans which will store the id of the loan with a boolean whether that loan is created. To simulate the real Loan Manager's isUserLoanActive() function which reverts if the loan has been created - a condition was added which reverts if the loan was added to the mapping.

## Proof of concept

## Proof of Concept

```

    function testLoanCreationAndDeposit() public {
        Messages.MessageParams memory params1 = Messages.MessageParams({
            adapterId: 5,
            returnAdapterId: 2324,
            receiverValue: 5_000e18,
            gasLimit: 100_000,
            returnGasLimit: 100_000
        });

        spokeCommon.createAccount(params1, "0xFIRSTACCOUNT", bytes32(0));
        spokeCommon.createAccount(params1, "0xSECONDACCOUNT", bytes32(0));

        //Since we're just "simulating the creation", we'll be manually adding the loans in the loan manager instead of going through the Spoke Common.

        //For the purpose of simplicity, just the createUserLoan is tested, instead of the createUserLoanAndDeposit, as the revert will happen on loan creation.

        //A Malicious user sees a createLoanAndDeposit transaciton in the mempool which they plan on frontrunning with the intent so that the transaction reverts on the hub. Since checks whether the loanId exists are performed on the hub, the message will be temporarily "stuck" together with the funds, until its reversed.

        //Malicious user:
        loanManager.createUserLoan("0xFIRSTLOAN", "0xSECONDACCOUNT", 12, "Malicious Loan");

        //Honest user transaction reverting:
        vm.expectRevert("Loan already exists");
        loanManager.createUserLoan("0xFIRSTLOAN", "0xFIRSTACCOUNT", 12, "First Loan");
    }
```
