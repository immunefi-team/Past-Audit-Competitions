# Boost \_ Folks Finance 33609 - \[Smart Contract - Medium] Account creation can be frontrun making the

Submitted on Wed Jul 24 2024 14:15:37 GMT-0400 (Atlantic Standard Time) by @Kalogerone for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33609

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0x3324B5BF2b5C85999C6DAf2f77b5a29aB74197cc>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A user who tries to create an account for the protocol has to choose his `accountId`. Any user can frontrun this transaction with the same `accountId`, making the initial user's transaction to revert because his selected `accountId` is taken.

## Vulnerability Details

Each account has a unique `bytes32` identifier named `accountId`. During the account creation, each user is asked to provide the `accountId` that his account will have.

```javascript
SpokeCommon.sol

    function createAccount(
        Messages.MessageParams memory params,
@>      bytes32 accountId,
        bytes32 refAccountId
    ) external payable nonReentrant {
        _doOperation(params, Messages.Action.CreateAccount, accountId, abi.encodePacked(refAccountId));
    }
```

This arbitrary `accountId` value is sent through a bridge to the `Hub.sol` contract which in turn calls the `createAccount` function is `AccountManager.sol`.

```javascript
Hub.sol

    function _receiveMessage(Messages.MessageReceived memory message) internal override {
        Messages.MessagePayload memory payload = Messages.decodeActionPayload(message.payload);
        .
        .
        .
        if (payload.action == Messages.Action.CreateAccount) {
            bytes32 refAccountId = payload.data.toBytes32(index);

@>          accountManager.createAccount(payload.accountId, message.sourceChainId, payload.userAddress, refAccountId);
        } else if
        .
        .
        .
    }
```

```javascript
AccountManager.sol

    function createAccount(
        bytes32 accountId,
        uint16 chainId,
        bytes32 addr,
        bytes32 refAccountId
    ) external override onlyRole(HUB_ROLE) {
        // check account is not already created (empty is reserved for admin)
@>      if (isAccountCreated(accountId) || accountId == bytes32(0)) revert AccountAlreadyCreated(accountId);
        .
        .
        .
    }
```

At this point, if there is already an account with the desired `accountId`, the transaction reverts. An attacker can take advantage of this and frontrun all the account creation transactions (on the chains with a public mempool, like the `Ethereum mainnet`) and prevent all the users from creating an account, which is essential for someone to use the protocol.

## Impact Details

This is a griefing attack which prevents any new users from using the protocol, since they can't create an account. Every transaction will fail because the attacker can frontrun it with the same `accountId`.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/spoke/SpokeCommon.sol#L27>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/Hub.sol#L163>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/AccountManager.sol#L42>

## Recommendation

Don't allow for the users to select their desired `accountId`. Use a counter internally and increment it with every account creation and use it as the `accountId`.

## Proof of concept

## Proof of Concept

Let's follow this scenario:

1. Bob tries to create an account with `accountId = "BOB_ACCOUNT_ID"`
2. Alice (the attacker) sees this transaction in the mempool and frontruns bob transaction with `accountId = "BOB_ACCOUNT_ID"`
3. Alice's transaction goes through
4. Bob's transaction gets reverted
5. Repeat

Add the following test in the `test/AccountManager.test.ts` file under the `describe("Create Account", () => {` tab.

```javascript
    it("Should frontrun account creation", async () => {

      const { admin, hub, unusedUsers, accountManager } = await loadFixture(deployAccountManagerFixture);
      const bob = unusedUsers[0];
      const alice = unusedUsers[1];
      const bobAddr = convertEVMAddressToGenericAddress(bob.address);
      const aliceAddr = convertEVMAddressToGenericAddress(alice.address);
      const accountId: string = getAccountIdBytes("BOB_ACCOUNT_ID");
      const spokeChainId = 0;
      const refAccountId: string = getEmptyBytes(BYTES32_LENGTH);
      console.log(accountId);

      const createAccountFrontrun = await accountManager
        .connect(hub)
        .createAccount(accountId, spokeChainId, aliceAddr, refAccountId);

      const createAccount = accountManager
        .connect(hub)
        .createAccount(accountId, spokeChainId, bobAddr, refAccountId);

      await expect(createAccount)
        .to.be.revertedWithCustomError(accountManager, "AccountAlreadyCreated")
        .withArgs(accountId);
    });
```
